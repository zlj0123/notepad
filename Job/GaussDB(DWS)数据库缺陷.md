
## 测试程序

![[Pasted image 20240927133241.png]]
![[Pasted image 20240927160621.png]]
![[Pasted image 20240927133353.png]]

## 测试结果
1. 如果只执行test2()方法，不管执行几次，都是正确的。也就是只执行大数据开发平台的数据预览，永远正确。![[Pasted image 20240927131825.png]]
2. 如果执行test1()方法后，又去执行test2()方法。也就是模拟执行一次数据源查询，然后再去大数据开发平台点击数据预览，报错。![[Pasted image 20240927132204.png]]
3. 如果执行test1()方法后，又去执行test2()方法两次，发现第二次又可以了，模拟一次数据源点击，二次大数据开发平台数据预览点击，第一次出错，第二次可以。![[Pasted image 20240927132759.png]]


## 结论

如上测试结果，得出结论，至少目前提供的GaussDB(DWS)数据库存在缺陷，不知道新的有没有问题，请联系华为确认。

## 附完整测试程序
maven坐标
```
<dependency>  
    <groupId>com.huaweicloud.dws</groupId>  
    <artifactId>huaweicloud-dws-jdbc</artifactId>  
    <version>8.3.1.200-200</version>  
</dependency>
```

测试程序
```
package io.ibigdata.hadoop.hdfs;  
  
import java.sql.*;  
  
public class GuassDBDWSTest {  
    public static void main(String[] args) throws SQLException, ClassNotFoundException {  
        test1();  
  
        try {  
            test2();  
        } catch (Exception e) {  
            System.out.println(e.getMessage());  
        }  
  
        // 执行test2()方法两次，模拟大数据开发平台里，离线同步任务的数据预览，点击第二次，又正确了。  
        test2();  
    }  
  
    /**  
     * test1()方法模拟数据源的数据预览。  
     * 数据源配置的url为jdbc:gaussdb://10.20.24.29:8000/postgres?currentSchema=data_source  
     * 然后在数据源的数据预览里，会传进来catalog和schema的参数，值为 postgres.data_source  
     * 我们会把这个值用split(.)去分隔，得到catalog和schema，然后把上面的url返乡查找最后一个斜杠(/)，  
     * 把后面的值替换成cata，url变为jdbc:gaussdb://10.20.24.29:8000/postgres，  
     * 然后创建出connection，然后再执行connection.setSchema("data_source")，  
     * 我们执行上面的setSchema方法后会通过connection.getSchema()得到schema，  
     * 这个值永远是我们预期的实际schema，为data_source  
     */    public static void test1() throws SQLException, ClassNotFoundException{  
        System.out.println("---------test1----------");  
        String driver = "com.huawei.gauss200.jdbc.Driver";  
        String jdbcURL = "jdbc:gaussdb://10.20.24.29:8000/postgres";  
        String username = "postgres_data_source";  
        String password = "123456";  
        String schema = "data_source";  
  
        Class.forName(driver);  
  
        String metaDataSql = "select * from muti_types limit 20";  
        try (Connection connection = DriverManager.getConnection(jdbcURL,username,password)) {  
            connection.setSchema(schema);  
  
            System.out.println("schema is :" + connection.getSchema());  
  
            Statement stmt = connection.createStatement();  
            ResultSet metaRs = stmt.executeQuery(metaDataSql);  
  
            while (metaRs.next()) {  
                System.out.println("get one record");  
            }  
        }  
  
    }  
  
  
    /**  
     * test2()方法模拟大数据开发平台里，离线同步任务的数据预览。  
     * 取的url为数据源里配置的jdbc url为，jdbc:gaussdb://10.20.24.29:8000/postgres?currentSchema=data_source  
     * 然后他这里没有catalog和schema传递过来，直接是一个url，我们直接用这个url创建一个connection，  
     * 我们也通过connection.getSchema()得到schema，并打印  
     * 但这里就很奇怪了，如果单独执行test2()方法，不管执行多少次，schema正确，是我们期待的data_source。  
     * 但是如果先执行test1()方法后，再执行test2()方法，发现取得的schema为pulbic，不正确，当然后面的select查询也报错了。  
     */  
    public static void test2() throws SQLException, ClassNotFoundException{  
        System.out.println("---------test2----------");  
        String driver = "com.huawei.gauss200.jdbc.Driver";  
        String jdbcURL = "jdbc:gaussdb://10.20.24.29:8000/postgres?currentSchema=data_source";  
        String username = "postgres_data_source";  
        String password = "123456";  
  
        Class.forName(driver);  
  
        String metaDataSql = "select * from muti_types limit 50";  
        try (Connection connection = DriverManager.getConnection(jdbcURL,username,password)) {  
            System.out.println("schema is :" + connection.getSchema());  
  
            Statement stmt = connection.createStatement();  
            ResultSet metaRs = stmt.executeQuery(metaDataSql);  
  
            while (metaRs.next()) {  
                System.out.println("get one record");  
            }  
        }  
    }  
}
```
