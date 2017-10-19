Config Class
@Configuration
@EnableBatchProcessing
@ComponentScan({"com.insight.web.batch.migration.usage"})
public class UsageConfiguration extends DefaultBatchConfigurer{

    @Autowired
    private JobBuilderFactory jobs;

    @Autowired
    private StepBuilderFactory steps;

//    @Bean()
//    @Primary
//    public DataSource webdbresetDatasource() {
//        DriverManagerDataSource dataSource = new DriverManagerDataSource();
//        dataSource.setDriverClassName("net.sourceforge.jtds.jdbc.Driver");
//        dataSource.setUrl("jdbc:jtds:sqlserver://HQIOSQLDEV08:14330/web_reporting;instanceName=APP09");
//        dataSource.setUsername("mxdeveloper");
//        dataSource.setPassword("develop2please");
//        return dataSource;
//    }

    @Bean(name = "updateUsageEntryJob")
    public Job job(Step step, UsageJobCompletionNotificationListener listener) throws Exception {
        return jobs.get("job")
                .incrementer(new RunIdIncrementer())
                .listener(listener)
                .start(step)
                .build();
    }

    @Bean
    public Step step(@Qualifier("pagingItemReader") ItemReader<UsageOrderMaintenance> reader,
                    UsageProcessor processor,
                    JdbcBatchItemWriter<UsageOrderMaintenance> writer){
            return steps.get("step")
                .<UsageOrderMaintenance,UsageOrderMaintenance>chunk(10000)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }


    @Bean
    public ItemReader<UsageOrderMaintenance> pagingItemReader(@Qualifier("webdbresetDatasource") DataSource dataSource,
                                                              UsageRowMapper rowMapper) {

        JdbcCursorItemReader<UsageOrderMaintenance> reader = new JdbcCursorItemReader<UsageOrderMaintenance>();
        String sql ="SELECT query here";
        reader.setSql(sql);
        reader.setDataSource(dataSource);
        reader.setRowMapper(rowMapper);
        return reader;
    }


    @Bean
    protected JdbcBatchItemWriter<UsageOrderMaintenance> writer(@Qualifier("webdbresetDatasource") DataSource dataSource) throws Exception {
        JdbcBatchItemWriter<UsageOrderMaintenance> itemWriter = new JdbcBatchItemWriter<UsageOrderMaintenance>();
        itemWriter.setDataSource(dataSource);
        itemWriter.setSql("INSERT query here)");
        itemWriter.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<UsageOrderMaintenance>());

        return itemWriter;
    }

    @Override
    @Autowired
    public void setDataSource(@Qualifier("webdbresetDatasource") DataSource dataSource) {
        super.setDataSource(dataSource);
    }

    @Bean
    JobLauncherTestUtils jobLauncherTestUtils(){
        return new JobLauncherTestUtils();
    }


    @Bean
    public JobRepository jobRepository(PlatformTransactionManager transactionManager) throws Exception {
        MapJobRepositoryFactoryBean jobRep = new MapJobRepositoryFactoryBean();
        jobRep.setTransactionManager(transactionManager);
        return jobRep.getObject();
    }

    /**
     * This is a launcher which is used launch the job when we are using the scheduler
     * @param jobRepository
     * @return
     * @throws Exception
     */
    @Bean(name = "usageJobLauncher")
    public JobLauncher jobLauncher(JobRepository jobRepository) throws Exception {
        SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
        jobLauncher.setJobRepository(jobRepository);

        return jobLauncher;
    }


}
Processor
@Component
public class UsageProcessor implements ItemProcessor<UsageOrderMaintenance, UsageOrderMaintenance> {

    @Autowired
    @Qualifier("usageDao")
    UsageDao usageDao;

    private EnvironmentProperty envProperty;

    private static final Logger LOGGER = Logger.getLogger(UsageProcessor.class);

    @Override
    public UsageOrderMaintenance process(UsageOrderMaintenance usageOrderMaintenance) throws Exception {
        final UsageOrderMaintenance orderMaintenance = new UsageOrderMaintenance();

        /** Get the values in the local fields */

        Date usagePeriod    =  usageOrderMaintenance.getUsagePeriod();
        Date createdOn      =  usageOrderMaintenance.getCreatedOn();
        int soldTo          =  usageOrderMaintenance.getPartnerNumber();
        int salesOrg        =  usageOrderMaintenance.getSalesOrg();
        int lineItemQty     =  usageOrderMaintenance.getLineItemQty();
        int lineItemNo      =  usageOrderMaintenance.getLineItemNumber();
        int maintCoverage   =  usageOrderMaintenance.getMaintCoverage();
        String usageType    =  usageOrderMaintenance.getUsageType();
        String enrollId     =  usageOrderMaintenance.getEnrollmentId();

        /** Only allow the UsageTypes that are available in the usage mapping of the particular salesOrg
         * It will be null if it is present
         */
        if(!StringUtils.isEmpty(usageType)) {
            /** Check if the usage data is already in the web database or not, if not then only insert the data in web */
            if(usageDao.checkIfRecordAlreadyExistsInUsageOrderMaintainence(String.valueOf(soldTo), String.valueOf(salesOrg), enrollId, usageType, usagePeriod)){
                orderMaintenance.setUsagePeriod(usagePeriod);
                orderMaintenance.setPartnerNumber(soldTo);
                orderMaintenance.setSalesOrg(salesOrg);
                orderMaintenance.setUsageType(usageType);
                orderMaintenance.setCreatedOn(createdOn);
                orderMaintenance.setLineItemQty(lineItemQty);
                orderMaintenance.setLineItemNumber(lineItemNo);
                orderMaintenance.setMaintCoverage(maintCoverage);
                orderMaintenance.setEnrollmentId(enrollId);
                LOGGER.info("Usage Received with Type ["+usageOrderMaintenance.getUsageType()+"]");
                return orderMaintenance;
            }else {
                LOGGER.info("Skipping the result. Its already recorded in the usage_table!!");
                return null;
                }
        }else {
            LOGGER.info("Not Applicable usageType for the given sales org. Sales Org: ["+salesOrg+"]");
            return null;
        }
    }
}

Row mapper
@Component
public class UsageRowMapper implements RowMapper<UsageOrderMaintenance> {

    private EnvironmentProperty envProperty;
    private CommonUtils commonUtils;

    @Override
    public UsageOrderMaintenance mapRow(ResultSet resultSet, int i) throws SQLException {
        String salesOrg =resultSet.getString("sales_organization");
        String programId = resultSet.getString("ZPROGRAM");

        String usageType = commonUtils.getUsageType(envProperty, programId, salesOrg);
        /**
         * If usageType cannot be found or is not present in the usage mapping then we are just making it empty String
         * to avoid the NULLPointException while passing the object to the processor.
         */
        if(StringUtils.isEmpty(usageType)){
            usageType = "";
        }
        //ZCLDSTRDT
        /** Change the usage date - setting day as the first day of month in the usage_maint_table */
        Date currentDate = resultSet.getDate("order_create_date");
        Date usageDate = new Date(currentDate.getYear(), currentDate.getMonth(), 1);

        Date createdDate = new Date();

        UsageOrderMaintenance orderMaintenance = new UsageOrderMaintenance();
        orderMaintenance.setSalesOrg(Integer.parseInt(salesOrg));
        orderMaintenance.setPartnerNumber(Integer.parseInt(resultSet.getString("customer_soldto")));
        orderMaintenance.setCreatedOn(createdDate);
        orderMaintenance.setUsagePeriod(usageDate);
        orderMaintenance.setUsageType(usageType);
        orderMaintenance.setLineItemNumber(0);
        orderMaintenance.setLineItemQty(0);
        orderMaintenance.setMaintCoverage(0);
        orderMaintenance.setEnrollmentId(resultSet.getString("ZAGREEMNO"));

        return orderMaintenance;
    }
}

Notification
@Component
public class UsageJobCompletionNotificationListener extends JobExecutionListenerSupport {
    private static final Logger  LOGGER = Logger.getLogger(UsageJobCompletionNotificationListener.class);

    @Override
    public void afterJob(JobExecution jobExecution) {
        LOGGER.info("Usage Update to the Web using Batch Job Finished at : "+System.currentTimeMillis()+"---------------------");
    }

    @Override
    public void beforeJob(JobExecution jobExecution) {
        LOGGER.info("Usage Update to the Web using Batch Job Started at : "+System.currentTimeMillis()+"-----------------------");
    }
}
