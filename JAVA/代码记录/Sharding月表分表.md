## 开始
因为数据量越来越大，分库分表几乎是每个大型系统都不得不进行的一项工作。本文则是一片从零开始的入门分表指南。
* 需求
数据按照时间月份分表，每月一张表，查询与更新针对于最近的两个月。


* sql
```
-- 逻辑表，每个月表都根据逻辑表生成
CREATE TABLE `month` (
  `id` varchar(50) NOT NULL,
  `nname` varchar(255) NOT NULL,
  `ccreated` datetime DEFAULT NULL,
  `updated` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

-- 月表
CREATE TABLE `month_201908` (
  `id` varchar(50) NOT NULL,
  `nname` varchar(255) NOT NULL,
  `ccreated` datetime DEFAULT NULL,
  `updated` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

CREATE TABLE `month_201909` (
  `id` varchar(50) NOT NULL,
  `nname` varchar(255) NOT NULL,
  `ccreated` datetime DEFAULT NULL,
  `updated` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

## Maven依赖
经过测试，支持springboot 2.0.X+与1.5.X+。
```xml
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<!--<version>5.1.42</version> -->
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-configuration-processor</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<!-- https://mvnrepository.com/artifact/io.shardingsphere/sharding-jdbc-spring-boot-starter -->
		<dependency>
			<groupId>io.shardingsphere</groupId>
			<artifactId>sharding-jdbc-spring-boot-starter</artifactId>
			<version>3.0.0</version>
		</dependency>
		<dependency>
			<groupId>cn.hutool</groupId>
			<artifactId>hutool-all</artifactId>
			<version>4.6.7</version>
		</dependency>
		<!-- https://mvnrepository.com/artifact/org.apache.commons/commons-lang3 -->
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-lang3</artifactId>
		</dependency>
		<!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
			<version>1.1.20</version>
		</dependency>

```

## 分片策略与分片算法的选择
### 分片策略说明
我这里选择的是适合入门标准分片策略（`StandardShardingStrategy`）。
* StandardShardingStrategy 标准分片策略
  
  提供对SQL语句中的=, IN和BETWEEN AND的分片操作支持。StandardShardingStrategy只支持单分片键，提供PreciseShardingAlgorithm和RangeShardingAlgorithm两个分片算法。PreciseShardingAlgorithm是必选的，用于处理=和IN的分片。RangeShardingAlgorithm是可选的，用于处理BETWEEN AND分片，如果不配置RangeShardingAlgorithm，SQL中的BETWEEN AND将按照全库路由处理。
* ComplexShardingStrategy 复合分片策略

  提供对SQL语句中的=, IN和BETWEEN AND的分片操作支持。ComplexShardingStrategy支持多分片键，由于多分片键之间的关系复杂，因此并未进行过多的封装，而是直接将分片键值组合以及分片操作符透传至分片算法，完全由应用开发者实现，提供最大的灵活度。
* InlineShardingStrategy 行表达式分片策略

  使用Groovy的表达式，提供对SQL语句中的=和IN的分片操作支持，只支持单分片键。对于简单的分片算法，可以通过简单的配置使用，从而避免繁琐的Java代码开发，如: t_user_$->{u_id % 8} 表示t_user表根据u_id模8，而分成8张表，表名称为t_user_0到t_user_7。
* HintShardingStrategy Hint分片策略

  通过Hint而非SQL解析的方式分片的策略。
* NoneShardingStrategy 不分片策略

  就是不分片。

### 分片算法说明
这里需要实现`PreciseShardingAlgorithm`算法与`RangeShardingAlgorithm`算法。
* PreciseShardingAlgorithm 精确分片算法
  
  用于处理使用单一键作为分片键的=与IN进行分片的场景。需要配合StandardShardingStrategy使用。
* RangeShardingAlgorithm 范围分片算法

  用于处理使用单一键作为分片键的BETWEEN AND进行分片的场景。需要配合StandardShardingStrategy使用。

* ComplexKeysShardingAlgorithm 复合分片算法

  用于处理使用多键作为分片键进行分片的场景，包含多个分片键的逻辑较复杂，需要应用开发者自行处理其中的复杂度。需要配合ComplexShardingStrategy使用。
* HintShardingAlgorithm Hint分片算法

  用于处理使用Hint行分片的场景。需要配合HintShardingStrategy使用。

### 分片算法实现
* 精确分片算法
```JAVA
import java.util.Collection;
import java.util.Date;
import cn.hutool.core.date.DateUtil;
import io.shardingsphere.api.algorithm.sharding.PreciseShardingValue;
import io.shardingsphere.api.algorithm.sharding.standard.PreciseShardingAlgorithm;

public class MyPreciseShardingAlgorithm implements PreciseShardingAlgorithm<Date> {
	private static String yearAndMonth = "yyyyMM";

	@Override
	public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Date> shardingValue) {
		StringBuffer tableName = new StringBuffer();
		tableName.append(shardingValue.getLogicTableName()).append("_")
				.append(DateUtil.format(shardingValue.getValue(), yearAndMonth));
		return tableName.toString();
	}
}
```

* 范围分片算法
```JAVA
import java.util.Collection;
import java.util.Date;
import java.util.LinkedHashSet;
import cn.hutool.core.date.DateUtil;
import io.shardingsphere.api.algorithm.sharding.RangeShardingValue;
import io.shardingsphere.api.algorithm.sharding.standard.RangeShardingAlgorithm;

public class MyRangeShardingAlgorithm implements RangeShardingAlgorithm<Date> {
	private static String yearAndMonth = "yyyyMM";

	/**
	 * 查询时查询最近两个月
	 */
	@Override
	public Collection<String> doSharding(Collection<String> availableTargetNames,
			RangeShardingValue<Date> shardingValue) {
		Collection<String> result = new LinkedHashSet<String>();
		// 获取当前的年月,优化点，可以优化为全局变量
		Date now = new Date();
		int end = Integer.valueOf(DateUtil.format(now, yearAndMonth));
		// 获取前一个月
		int start = Integer.valueOf(DateUtil.format(DateUtil.lastMonth(), yearAndMonth));
		result.add(shardingValue.getLogicTableName() + "_" + start);
		result.add(shardingValue.getLogicTableName() + "_" + end);
		return result;
	}
}
```

## 配置

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    url: jdbc:mysql://localhost:3306/ddssss
    username: root
    password: youpan
    tomcat:
      initial-size: 5
    driver-class-name: com.mysql.jdbc.Driver
  jpa:
    database: mysql
sharding:
  jdbc:
    datasource:
      names: month-0
      month-0:
        driver-class-name: com.mysql.jdbc.Driver
        url: jdbc:mysql://localhost:3306/ddssss
        username: root
        password: youpan
        type: com.alibaba.druid.pool.DruidDataSource
    config:
      sharding:
        tables:
          month: 
            key-generator-column-name: id
            table-strategy:
              standard:
                sharding-column: ccreated
                precise-algorithm-class-name: com.example.sharding.config.MyPreciseShardingAlgorithm
                range-algorithm-class-name: com.example.sharding.config.MyRangeShardingAlgorithm
        props:
          sql.show: true
```

## 测试
```JAVA

import java.util.Date;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import com.example.sharding.domain.Month;

import cn.hutool.json.JSONUtil;
import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class StartRunner implements CommandLineRunner {
	@Autowired
	MonthService monthService;

	@Override
	public void run(String... args) throws Exception {
		System.out.println("==============init===================");
		Month month = new Month();
		month.setNname("ssss");
		month.setCcreated(new Date());
		month.setUpdated(new Date());
		// 添加记录
		monthService.save(month);
		log.info("add month:{}", JSONUtil.parse(month));
		// 查询记录
		Month existMonth = monthService.select(month.getId());
		log.info("find existMonth:{}", JSONUtil.parse(existMonth));
		// 修改记录
		month.setNname("updated");
		monthService.update(month);
		existMonth = monthService.select(month.getId());
		log.info("find existMonth:{}", JSONUtil.parse(month));

	}

}
```
启动后就会看到日志如下：


