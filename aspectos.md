# Aspectos adicionales y ejercicio final

## Algunos aspectos adicionales

Como se comentó en las diapositivas de clase, existen algunos aspectos adicionales (uso de combinadores, particiones, etc...), que nos pueden servir para mejorar las prestaciones de los procesos *MapReduce*. Tienes una guía breve en el [tutorial Hadoop de Cloudera](http://www.cloudera.com/content/cloudera/en/documentation/hadoop-tutorial/CDH5/Hadoop-Tutorial/ht_mapreduce_if.html). Lee tranquilamente dicha guía para entender mejor estos aspectos.

### WordCount V2: configuraciones específicas

Este primer ejemplo está extraído del [tutorial Hadoop de Cloudera](http://www.cloudera.com/content/cloudera/en/documentation/hadoop-tutorial/CDH5/Hadoop-Tutorial/ht_mapreduce_if.html), mencionado anteriormente.
Vamos a utilizar las opciones de configuración de Hadoop para incorporar un parámetro que nos permita elegir si el conteo de palabras va a ser sensible (o no) a mayúsculas. Además, haremos uso de un **combiner** que va a combinar los pares `<clave,valor>` que sean locales al **map** utilizado (incremento de prestaciones).

Consulta el fichero [WordCount.java](code/ejemplo4/WordCount.java). Se han realizado los siguientes cambios:

* Importamos la clase `Configuration`. Se puede utilizar esta clase para acceder a argumentos de la línea de comandos en tiempo de ejecución:
```java
import org.apache.hadoop.conf.Configuration;
```
* Creamos una variable para establecer (o no) si el **map** va a ser sensible a mayúsculas:
```java
private boolean sensibleMayusculas = false;
```
* Añadimos un método `setup`. Hadoop llama a este método automáticamente al mandar un trabajo. Este código instancia un objeto de tipo `Configuration`, y después establece el valor de la variable `sensibleMayusculas` al valor de la variable de sistema  `wordcount.mayusculas.sensible` que se supone que ha sido especificada por línea de comandos (valor por defecto `false`).
```java
    protected void setup(Mapper.Context context)
      throws IOException,
        InterruptedException {
      Configuration config = context.getConfiguration();
      this.sensibleMayusculas = config.getBoolean("wordcount.mayusculas.sensible", false);
    }
```
* Desactivamos la sensibilidad a mayúsculas aquí. Si `sensibleMayusculas` es `false`, la línea completa se convierte a minúsculas antes de que sea procesado por el `StringTokenizer`:
```java
      if (!sensibleMayusculas) {
        line = line.toLowerCase();
      }
```
* Además, puedes observar como, en este caso, se ha utilizado el mismo *reducer* para servir de combinador de pares locales:
```java
    job.setCombinerClass(MiReduce.class);
```
Compila el ejemplo como hiciste anteriormente. Ahora podrás ejecutar el ejemplo de dos formas. Por defecto, el programa no será sensible a mayúsculas:
```bash
hadoop jar wordcount.jar master.sd.WordCount input output
```

Si queremos que sea sensible a mayúsculas podemos utilizar:
```bash
hadoop jar wordcount.jar master.sd.WordCount -Dwordcount.mayusculas.sensible=true input output
```

### MostFrequentNextWord

El fichero [MostFrequentNextWord.java](code/ejemplo5/MostFrequentNextWord.java) muestra una versión del contador de palabras, en la que establecemos cuál es la palabra que aparece con más frecuencia detrás de cada palabra. Para ello hacemos un encadenamiento (*chaining*) de dos procesos *MapReduce*, de forma que el primer proceso obtiene el conteo de todas las parejas de palabras que aparecen seguidas y, utilizando el resultado de este conteo, el segundo analiza, para cada palabra, cuál es la palabra que aparece después con más frecuencia.

La forma en que se ejecutan dos procesos encadenados de *MapReduce* es utilizar el directorio de salida del primero como directorio de entrada del primero. Lo único que tenemos que hacer es configurar el método `run` para que haya dos trabajos (dos objetos de la clase `Job`) y especificar un par de clases *MapReduce* para cada trabajo (en nuestro caso, `MiMap`, `MiReduce`, `MiMap2` y `MiReduce2`):
```java
	public int run(String[] args) throws Exception {
		Job job = Job.getInstance(getConf(), "mostFrequentNextWord-MR1");
		job.setJarByClass(this.getClass());
		FileInputFormat.addInputPath(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));
		job.setMapperClass(MiMap.class);
		job.setReducerClass(MiReduce.class);
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);
		if (! job.waitForCompletion(true))
			return 1;

		Job job2 = Job.getInstance(getConf(), "mostFrequentNextWord-MR2");
		job2.setJarByClass(this.getClass());
		FileInputFormat.addInputPath(job2, new Path(args[1]));
		FileOutputFormat.setOutputPath(job2, new Path(args[2]));
		job2.setMapperClass(MiMap2.class);
		job2.setReducerClass(MiReduce2.class);
		job2.setOutputKeyClass(Text.class);
		job2.setOutputValueClass(Text.class);
		return job2.waitForCompletion(true) ? 0 : 1;
	}
```
Observemos ahora cada uno de los procesos *MapReduce*. El primer **map** realiza lo siguiente:
```java
	public static class MiMap extends Mapper<LongWritable, Text, Text, IntWritable> {
		private final static IntWritable one = new IntWritable(1);
		private Text word = new Text();
		private static final Pattern WORD_BOUNDARY = Pattern.compile("\\s+");
		private static final Pattern NON_ALPHA = Pattern.compile("^.*[^a-zA-Z0-9].*$");

		public void map(LongWritable offset, Text lineText, Context context)
				throws IOException, InterruptedException {
			String line = lineText.toString();
			String[] words = WORD_BOUNDARY.split(line);
			for (int i=0; i<(words.length-1); i++) {
				if( !(words[i].isEmpty()) && !(words[i+1].isEmpty())){
					if (!NON_ALPHA.matcher(words[i]).matches() && !NON_ALPHA.matcher(words[i+1]).matches()) {
						this.word.set(words[i] + "-" + words[i+1]);
						context.write(this.word,one);
					}
				}
			}
		}
	}
```
Como puedes observar, hemos vuelto a dividir la línea en palabras. Luego recorremos una a una las palabras de la línea y observamos si la palabra actual y la siguiente contienen únicamente caracteres alfabéticos. Para esta tarea, nos valemos de la expresión regular `"^.*[^a-zA-Z0-9].*$"` que buscaría cualquier carácter no alfabético. Finalmente, escribimos en el objeto `Context` la pareja de palabras separadas por `-` como la clave y un uno para indicar que aparecen una vez.

Por su parte, el primer **reduce** simplemente va a combinar los pares `<clave,valor>` generados por el primer **map**, acumulando el número de ocurrencias de esa pareja de palabras:
```java
	public static class MiReduce extends Reducer<Text, IntWritable, Text, IntWritable> {
		@Override
		public void reduce(Text word, Iterable<IntWritable> counts, Context context)
				throws IOException, InterruptedException {
			int sum = 0;
			for (IntWritable count : counts) {
				sum += count.get();
			}
			context.write(word, new IntWritable(sum));
		}
	}
```

La salida del primer proceso **Reduce** se va a generar en el directorio `new Path(args[1])`. Recuerda que la salida generada siempre incluye la clave (en este caso la pareja de palabras separadas por un `-`), un carácter tabulador y el valor (en este caso, el número de veces que aparece la pareja en el texto). Ahora, el segundo proceso **map** va a crear un conjunto de pares `<clave,valor>` dónde, como clave, tendremos cada una de las palabras y, como valor, tendremos una cadena que incluirá la palabra que aparece después, un guión `-` y el número de ocurrencias:
```java
	public static class MiMap2 extends Mapper<LongWritable, Text, Text, Text> {

		public void map(LongWritable offset, Text lineText, Context context)
				throws IOException, InterruptedException {
			String line = lineText.toString();

			int index, index2;
			if((index = line.indexOf('-')) == -1)
				return;
			if((index2 = line.indexOf('\t')) == -1)
				return;

			String word = line.substring(0, index);
			String nextWord = line.substring(index+1, index2);
			String repetitions = line.substring(index2+1);
			context.write(new Text(word), new Text(nextWord + '-' + repetitions));
		}
	}
```

Todos estos pares `<clave,valor>` se combinarán y mandarán al segundo **reducer**, que va a buscar cuál es la palabra más frecuente (mayor número de ocurrencias) iterando sobre la lista correspondiente al conjunto de valores de cada clave:
```java
	public static class MiReduce2 extends Reducer<Text, Text, Text, Text> {

		public void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
			String mostCommonNextWord = "";
			int maximum = 0;
			// Buscamos la palabra más común
			for (Text val : values) {
				String val_str = val.toString();
				int index;
				if((index = val_str.indexOf('-')) == -1)
					return;

				int occurrences = Integer.parseInt(val_str.substring(index+1));
				if( maximum < occurrences){
					mostCommonNextWord = val_str.substring(0, index);
					maximum = occurrences;
				}
			}
			if(maximum!=0)
				context.write(key, new Text(mostCommonNextWord));
		}  
	}
```
La ejecución es similar a la de otros ejemplos. Primero, borramos la carpeta de salida:
```bash
[cloudera@quickstart ejemplo5]$ hadoop fs -rm -r /user/cloudera/output
15/06/07 10:26:47 INFO fs.TrashPolicyDefault: Namenode trash configuration: Deletion interval = 0 minutes, Emptier interval = 0 minutes.
Deleted /user/cloudera/output
```
Después, creamos el `jar` (por ejemplo, utilizando Eclipse) y, una vez creado, lo lanzamos con el siguiente comando, dónde especificamos el directorio de entrada, el intermedio y el directorio de salida:
```bash
[cloudera@quickstart ejemplo5]$ hadoop jar mostFrequentNextWord.jar master.sd.MostFrequentNextWord input intermedio output
15/06/07 10:28:06 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
15/06/07 10:28:08 INFO input.FileInputFormat: Total input paths to process : 1
15/06/07 10:28:08 INFO mapreduce.JobSubmitter: number of splits:1
15/06/07 10:28:08 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1433695857588_0001
15/06/07 10:28:09 INFO impl.YarnClientImpl: Submitted application application_1433695857588_0001
15/06/07 10:28:09 INFO mapreduce.Job: The url to track the job: http://quickstart.cloudera:8088/proxy/application_1433695857588_0001/
15/06/07 10:28:09 INFO mapreduce.Job: Running job: job_1433695857588_0001
15/06/07 10:28:32 INFO mapreduce.Job: Job job_1433695857588_0001 running in uber mode : false
15/06/07 10:28:32 INFO mapreduce.Job:  map 0% reduce 0%
15/06/07 10:28:44 INFO mapreduce.Job:  map 67% reduce 0%
15/06/07 10:28:45 INFO mapreduce.Job:  map 100% reduce 0%
15/06/07 10:28:54 INFO mapreduce.Job:  map 100% reduce 100%
15/06/07 10:28:55 INFO mapreduce.Job: Job job_1433695857588_0001 completed successfully
15/06/07 10:28:55 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=7560110
		FILE: Number of bytes written=15340925
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=5590012
		HDFS: Number of bytes written=2317892
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=11089
		Total time spent by all reduces in occupied slots (ms)=7058
		Total time spent by all map tasks (ms)=11089
		Total time spent by all reduce tasks (ms)=7058
		Total vcore-seconds taken by all map tasks=11089
		Total vcore-seconds taken by all reduce tasks=7058
		Total megabyte-seconds taken by all map tasks=11355136
		Total megabyte-seconds taken by all reduce tasks=7227392
	Map-Reduce Framework
		Map input records=124787
		Map output records=479606
		Map output bytes=6600892
		Map output materialized bytes=7560110
		Input split bytes=126
		Combine input records=0
		Combine output records=0
		Reduce input groups=173394
		Reduce shuffle bytes=7560110
		Reduce input records=479606
		Reduce output records=173394
		Spilled Records=959212
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=223
		CPU time spent (ms)=6270
		Physical memory (bytes) snapshot=524898304
		Virtual memory (bytes) snapshot=3130150912
		Total committed heap usage (bytes)=434110464
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters
		Bytes Read=5589886
	File Output Format Counters
		Bytes Written=2317892
15/06/07 10:28:56 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
15/06/07 10:28:56 INFO input.FileInputFormat: Total input paths to process : 1
15/06/07 10:28:56 INFO mapreduce.JobSubmitter: number of splits:1
15/06/07 10:28:56 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1433695857588_0002
15/06/07 10:28:56 INFO impl.YarnClientImpl: Submitted application application_1433695857588_0002
15/06/07 10:28:56 INFO mapreduce.Job: The url to track the job: http://quickstart.cloudera:8088/proxy/application_1433695857588_0002/
15/06/07 10:28:56 INFO mapreduce.Job: Running job: job_1433695857588_0002
15/06/07 10:29:03 INFO mapreduce.Job: Job job_1433695857588_0002 running in uber mode : false
15/06/07 10:29:03 INFO mapreduce.Job:  map 0% reduce 0%
15/06/07 10:29:10 INFO mapreduce.Job:  map 100% reduce 0%
15/06/07 10:29:18 INFO mapreduce.Job:  map 100% reduce 100%
15/06/07 10:29:18 INFO mapreduce.Job: Job job_1433695857588_0002 completed successfully
15/06/07 10:29:18 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=2664686
		FILE: Number of bytes written=5550069
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=2318026
		HDFS: Number of bytes written=237140
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=4557
		Total time spent by all reduces in occupied slots (ms)=5352
		Total time spent by all map tasks (ms)=4557
		Total time spent by all reduce tasks (ms)=5352
		Total vcore-seconds taken by all map tasks=4557
		Total vcore-seconds taken by all reduce tasks=5352
		Total megabyte-seconds taken by all map tasks=4666368
		Total megabyte-seconds taken by all reduce tasks=5480448
	Map-Reduce Framework
		Map input records=173394
		Map output records=173394
		Map output bytes=2317892
		Map output materialized bytes=2664686
		Input split bytes=134
		Combine input records=0
		Combine output records=0
		Reduce input groups=19466
		Reduce shuffle bytes=2664686
		Reduce input records=173394
		Reduce output records=19466
		Spilled Records=346788
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=97
		CPU time spent (ms)=4570
		Physical memory (bytes) snapshot=499331072
		Virtual memory (bytes) snapshot=3131473920
		Total committed heap usage (bytes)=425197568
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters
		Bytes Read=2317892
	File Output Format Counters
		Bytes Written=237140
```
Finalmente, podemos comprobar la salida generada con el comando:
```bash
[cloudera@quickstart ejemplo5]$ hadoop fs -cat /user/cloudera/output/part*
```
## Ejercicio 3

* **Objetivo**: Escribe un programa *MapReduce* para Hadoop que implemente un algoritmo de recomendación simple para una red social del tipo "Personas que podrías conocer". La idea fundamental es que si dos personas tienen muchos amigos en común, entonces el sistema debería recomendarles ser amigos.
* **Fichero de entrada**: Descarga el fichero de [entrada](code/ejercicio3/ejercicio3Datos.zip). El fichero contiene la lista de adyacencia representada por una secuencia de líneas con el siguiente formato:

    ```
    <IDUsuario><\t><ListaAmigos>
    ```

    dónde `<IDUsuario>` es un entero correspondiente a un único usuario, `<\t>` es un carácter tabulador  y `<ListaAmigos>` es una lista separada por comas de los IDs de los amigos de `<IDUsuario>`. Hágase notar que los amigos son mutuos (es decir, las aristas son no dirigidas): si A es amigo de B, entonces B es amigo de A. Los datos proporcionados son consistentes con esta regla, habiendo un entrada por cada dirección de la arista.

* **Algoritmo**: Vamos a utilizar un algoritmo simple que, por cada usuario `U`, recomiende `N=10` usuarios que no sean amigos de `U` pero que tengan el mayor número de amigos en común con `U`.
* **Salida**: la salida generada por el programa debería contener una línea por usuario con el siguiente formato:

    ```
    <IDUsuario><\t><Recomendaciones>
    ```

    dónde `<IDUsuario>` es el identificador del usuario al que se le hacen las recomendaciones y `<Recomendaciones>` es una lista separada por comas de los 10 amigos recomendados por el algoritmo, ordenada en orden decreciente de número de amigos en común. Si un usuario no tiene recomendaciones, no introduzcas nada en la lista. Si hay usuarios recomendados con el mismo número de amigos en común, resuelve el empate utilizando orden creciente de sus identificadores.



## Referencias

Este tutorial se ha realizado basándonos en gran medida en los siguientes tutoriales:

1. [Introducción a la programación MapReduce en Hadoop](http://laurel.datsi.fi.upm.es/docencia/asignaturas/ppd). Universidad Politécnica de Madrid (UPM).
2. [Hadoop Tutorial](http://web.stanford.edu/class/cs246/homeworks/tutorial.pdf) Stanford University.
