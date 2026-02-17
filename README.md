 # Spring Batch is a specialized tool (a framework) for Java designed to process huge amounts of data in one go
## Imagine you have a million customer records in a CSV file that need to be cleaned and moved to a database. You wouldn't do this one-by-one, nor would you do it manually.

## Spring Batch is specifically designed for batch processing, allowing the execution of a series of steps without manual intervention, often in the background. It can be used for tasks such as processing large datasets,

## Spring Batch is your factory setup:
 ### Reader (The Loader): Takes 100 records off the truck (file) at a time.
### Processor (The Worker): Cleans/transforms those 100 records (e.g., formats dates, fixes names).
### Writer (The Stacker): Takes the cleaned 100 records and saves them into the database. 

Where to Use the Spring Batch?
Spring Batch is particularly well-suited for scenarios that involve large-scale data processing. Here are some common use cases:

Data Migration: Moving large volumes of data from one system to another, such as during a system upgrade or migration to a new database.
Report Generation: Creating reports from large datasets, which often involves reading data, processing it, and writing it to files or databases.
Data Integration: Combining data from multiple sources (like databases, APIs, or flat files) into a unified format for analytics or storage.
ETL Processes: Performing Extract, Transform, Load (ETL) operations, where data is extracted from various sources, transformed to meet business requirements, and loaded into a target system.
Batch Processing of Transactions: Processing financial transactions or logs in bulk to perform analytics or generate summaries.
Scheduled Jobs: Running jobs at specific times or intervals, such as nightly data updates, backups, or maintenance tasks.
Handling High Volume Transactions: When applications need to handle high volumes of transactions without affecting the performance of online systems.

# In your favorite IDE, create a new Maven-based Java 17+ project and add the following dependency to your pom.xml:
<dependencies>
    <dependency>
        <groupId>org.springframework.batch</groupId>
        <artifactId>spring-batch-core</artifactId>
        <version>6.0.2</version>
    </dependency>
</dependencies>


# Key Concepts of the Spring Batch
# 1. Job
The job represents the batch processing the pipeline. It consists of the multiple steps which are executed in the sequence.

@Bean
public Job importUserJob(JobBuilderFactory jobBuilderFactory, Step step1) {
    return jobBuilderFactory.get("importUserJob") // Create a job named "importUserJob"
                            .start(step1)         // Define the first step in the job
                            .build();             // Build the job
}

Explanation:

JobBuilderFactory: it creates the job instance.
start(step1): It defines the first step in the job.
# 2. Step
A Step is an individual phase of the job. Each step follows the defined sequence that are reading data, processing it, and writing it out
Each step can encapsulates the ItemReader, ItemProcessor, and ItemWriter

@Bean
public Step step1(StepBuilderFactory stepBuilderFactory, 
                  ItemReader<User> reader, 
                  ItemProcessor<User, ProcessedUser> processor, 
                  ItemWriter<ProcessedUser> writer) {

    return stepBuilderFactory.get("step1") // Create a step named "step1"
                             .<User, ProcessedUser>chunk(10) // Process 10 items at a time
                             .reader(reader)                   // Set the item reader
                             .processor(processor)             // Set the item processor
                             .writer(writer)                   // Set the item writer
                             .build();                         // Build the step
}
Explanation:

StepBuilderFactory: It creates the step instance.
.chunk(10): It defines that data will be processor, and the ItemWriter for this step.

# 3) ItemReader
The ItemReader is responsible for reading the input data, which could come from files, databases, or other sources.
@Bean
public FlatFileItemReader<User> reader() {
    return new FlatFileItemReaderBuilder<User>() // Builder for creating FlatFileItemReader
           .name("userItemReader")                // Set a name for the reader
           .resource(new ClassPathResource("users.csv")) // Specify the resource (CSV file)
           .delimited()                           // Specify that the file is delimited
           .names(new String[] {"id", "name", "email"}) // Define the field names
           .fieldSetMapper(new BeanWrapperFieldSetMapper<User>() {{ 
               setTargetType(User.class);          // Map fields to the User class
           }})
           .build();                              // Build the reader
}
Explanation:

FlatFileItemReaderBuilder: It creates the reader for the reading flat files like CSV.
resource(): It points to the file location.
names(): It defines the fields in the CSV file.
fieldSetMapper(): It maps the fields to the user class.


# 4) ItemProcessor 
The ItemProcessor transforms the input data into the output data. It applies the business logic such as the filtering, enriching, or converting the data.

public class UserItemProcessor implements ItemProcessor<User, ProcessedUser> {

    @Override
    public ProcessedUser process(final User user) throws Exception {
        String processedEmail = user.getEmail().toUpperCase(); // Convert email to uppercase
        return new ProcessedUser(user.getId(), user.getName(), processedEmail); // Create and return a processed user
    }
}

Explanation:

The process() method receives the User object and returns the ProcessedUser object after applying the transformation (converting the email to uppercase in this case).

# 5. ItemWriter
The ItemWriter writes the processed data to the desired output such as the file, database, or message queue.
Example (Writing to a database):

@Bean
public JdbcBatchItemWriter<ProcessedUser> writer(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<ProcessedUser>() // Builder for JdbcBatchItemWriter
           .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>()) // Maps properties to SQL parameters
           .sql("INSERT INTO processed_user (id, name, email) VALUES (:id, :name, :email)") // SQL query for insertion
           .dataSource(dataSource) // Set the data source
           .build(); // Build the writer
}
Explanation:

JdbcBatchItemWriterBuilder: It builds the writer for the batch database operations.
itemSqlParameterSourceProvider(): It maps the properties of the ProcessedUser to the SQL parameters.
sql(): It defines the SQL query to the insert the processed data into the database.

# 6. JobRepository
The JobRepository stores the job and step execution metadata, such as the execution history, job parameters, and the status of the job execution. It allows the Spring Batch to restart the jobs from the last committed point in the case of failure.

@Bean
public JobRepository jobRepository(DataSource dataSource, PlatformTransactionManager transactionManager) throws Exception {
    return new JobRepositoryFactoryBean() // Create a JobRepositoryFactoryBean instance
           .setDataSource(dataSource) // Specify the data source for storing job details
           .setTransactionManager(transactionManager) // Set the transaction manager
           .getObject(); // Retrieve the JobRepository instance
}
Explanation:

JobRepositoryFactoryBean: It configures the job repository to the store metadata.
dataSource(): It specifies the data source for storing the job details.

# 7. JobLauncher
The JobLauncher is responsible for the triggering jobs. We can start the jobs programmatically or use schedular to run jobs periodically.
Example:

@Autowired
private JobLauncher jobLauncher; // Autowired JobLauncher

@Autowired
private Job job; // Autowired Job

public void runJob() {
    try {
        JobParameters params = new JobParametersBuilder() // Create job parameters
                .addLong("time", System.currentTimeMillis()) // Add a timestamp parameter
                .toJobParameters(); // Build the job parameters
        jobLauncher.run(job, params); // Launch the job with parameters
    } catch (Exception e) {
        e.printStackTrace(); // Handle exceptions during job execution
    }
}
Explanation:

JobLauncher: It executes the job.
JobParameters: It supplies parameters to the uniquely identify job instances, ensuring that the same job can be run multiple times with different inputs.

#  Chunk-Oriented Processing
Spring Batch's chunk-oriented processing is the pattern where data can be read, processed, and written in the chunks. Each chunk can be treated as the single transaction, ensuring the reliability and restartability.

In the following example, a step processes 10 items at the time

@Bean
public Step step(StepBuilderFactory stepBuilderFactory, 
                 ItemReader<User> reader, 
                 ItemProcessor<User, ProcessedUser> processor, 
                 ItemWriter<ProcessedUser> writer) {

    return stepBuilderFactory.get("step") // Create a step named "step"
                             .<User, ProcessedUser>chunk(10) // Process 10 items at a time
                             .reader(reader) // Set the item reader
                             .processor(processor) // Set the item processor
                             .writer(writer) // Set the item writer
                             .build(); // Build the step
}
Explanation:

The chunk size (chunk(10)) ensures that 10 records are processed and committed in each transaction.


# Features of the Spring Batch
Declarative I/O operations: Easily read and write data from various sources like CSV, XML, databases, etc.
Transaction management: Ensures consistent processing with built-in transaction management.
Job Restartability: Supports restarting jobs from where they left off after failure.
Error handling and retry: Built-in mechanisms for handling failures, retrying operations, and skipping faulty records.
Scheduling: Spring Batch can be integrated with Quartz or Spring’s @Scheduled to schedule batch jobs.
