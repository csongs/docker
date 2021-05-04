# 範例
進入網址: http://127.0.0.1:7700/
## http 範例
- tech.powerjob.official.processors.impl.HttpProcessor
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

## sql範例
- tech.powerjob.official.processors.impl.sql.DynamicDatasourceSqlProcessor
- 任務參數
```
{
	"sql": "SELECT * FROM `powerjob-product`.app_info;",
	"jdbcUrl": "jdbc:mysql://172.27.0.2:3306/powerjob-product?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Taipei",
	"timeout": "30"

}
```
- 其他
```
Worker 要配置開啟功能 -Dpowerjob.official-processor.dynamic-datasource.enable=true
```
