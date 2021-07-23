# 範例
- 進入網址: http://127.0.0.1:7700/
- powerjob 介紹: https://github.com/PowerJob/PowerJob
## http 範例
- 執行位置: `tech.powerjob.official.processors.impl.HttpProcessor`
- 任務參數
```
{
	"method": "GET",
	"timeout": "30",
	"url": "https://google.com",
	"headers": {
		"Authorization": "header_example"
	}
}
```

## sql-連線資訊當作參數
- 執行位置: `tech.powerjob.official.processors.impl.sql.DynamicDatasourceSqlProcessor`
- 任務參數
```
{
	"sql": "SELECT * FROM `powerjob-product`.app_info;",
	"jdbcUrl": "jdbc:mysql://172.27.0.2:3306/powerjob-product?usrename=secret&password=secret&useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Taipei",
	"timeout": "30"

}
```
- 其他
```
Worker 要配置開啟功能 -Dpowerjob.official-processor.dynamic-datasource.enable=true
```
## sql-連線參數在worker
- 使用內部類別
- 執行位置: `tech.powerjob.official.processors.impl.sql.SpringDatasourceSqlProcessor`
- 使用參數EX1
```
{
	"dataSourceName": "default",
	"sql": "select 'x' from DUAL",
	"timeout": 10,
	"showResult": true
}
```
- 使用參數EX2
```
{
	"dataSourceName": "CPQ",
	"sql": "SELECT *  FROM dbo.table ",
	"timeout": 10,
	"showResult": true
}
```
- java 範例
```java
package tech.powerjob.samples.config;

import tech.powerjob.common.utils.CommonUtils;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.h2.Driver;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DependsOn;
import org.springframework.context.annotation.Primary;

import tech.powerjob.official.processors.impl.sql.SpringDatasourceSqlProcessor;
import tech.powerjob.samples.util.UtilConf;

import javax.sql.DataSource;

@Configuration
public class SqlProcessorConfiguration {

    @Autowired
    UtilConf utilConf;

    @Bean
    @DependsOn({"initPowerJob"})
    public DataSource sqlProcessorDataSource() {
        String path = System.getProperty("user.home") + "/test/h2/" + CommonUtils.genUUID() + "/";
        String jdbcUrl = String.format("jdbc:h2:file:%spowerjob_sql_processor_db;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false", path);
        HikariConfig config = new HikariConfig();
        config.setDriverClassName(Driver.class.getName());
        config.setJdbcUrl(jdbcUrl);
        config.setAutoCommit(true);
        // 池中最小空闲连接数量
        config.setMinimumIdle(1);
        // 池中最大连接数量
        config.setMaximumPoolSize(10);
        return new HikariDataSource(config);
    }


    @Bean
    public SpringDatasourceSqlProcessor simpleSpringSqlProcessor(@Qualifier("sqlProcessorDataSource") DataSource dataSource) {
        SpringDatasourceSqlProcessor springDatasourceSqlProcessor = new SpringDatasourceSqlProcessor(dataSource);
        // do nothing
        springDatasourceSqlProcessor.registerSqlValidator("fakeSqlValidator", sql -> true);
        // 排除掉包含 drop 的 SQL
        springDatasourceSqlProcessor.registerSqlValidator("interceptDropValidator", sql -> sql.matches("^(?i)((?!drop).)*$"));
        // do nothing
        springDatasourceSqlProcessor.setSqlParser((sql, taskContext) -> sql);
        
        // 增加額外資料源
        springDatasourceSqlProcessor.registerDataSource("CPQ", cpqDataSource());
        
        return springDatasourceSqlProcessor;
    }
    
    // 增加另一個數據源
    public DataSource cpqDataSource() {
        //return DataSourceBuilder.create().build();
    	return DataSourceBuilder
    	        .create()
    	        .username(utilConf.getUserName())
    	        .password(utilConf.getPassword())
    	        .url(utilConf.getUrl())
    	        .driverClassName("oracle.jdbc.driver.OracleDriver")
    	        .build();
    }
    

 

}
```

## udf-自訂義job
- 引用共用lib其實有很多衝突,花很多時間再處理
- 一樣可以類似framwork專案,建立好 maven module ,讓大家更專心在處理商業邏輯上
- log要測試確認是否能轉到ELK
- 部署K8S 目前是寫死IP,不利於動態擴展 (SERVER掛掉重新部署可能會替換IP)
- 執行位置: 自訂義位置 `tech.powerjob.samples.pl.SampleJob`

- java
```java
package tech.powerjob.samples.pl;

import java.util.Date;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.stereotype.Component;

import tech.powerjob.samples.pl.util.Const;
import tech.powerjob.worker.core.processor.ProcessResult;
import tech.powerjob.worker.core.processor.TaskContext;
import tech.powerjob.worker.core.processor.sdk.BasicProcessor;
import tech.powerjob.worker.log.OmsLogger;

@Component
public class SampleJob extends GdsDAO implements BasicProcessor {
    private static final long serialVersionUID = 5494783169229434004L;
    private static Log log = LogFactory.getLog(SyncExpenseFromBISJob.class);
    
    
    @Override
    public ProcessResult process(TaskContext context) throws Exception {
    	
    	OmsLogger omsLogger = context.getOmsLogger();
    	omsLogger.info("run():" + (new Date()));
    	
    	// 取得參數使用
    	omsLogger.info("param():" + CommonUtils.parseParams(context));
        
    	//excute job
	// ...
        
        omsLogger.info("run() end:" + (new Date()));
        omsLogger.info(admMsg.toString());
        return new ProcessResult(success, context + ": " + success);
    }

}
```
