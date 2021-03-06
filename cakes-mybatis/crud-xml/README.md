# crud-xml

---

### 1.初始化
* 新建工程并导入gradle依赖
```groovy
dependencies {
    compile('org.mybatis:mybatis:3.5.1')
    compile('mysql:mysql-connector-java:8.0.13')
}
```

* SQL脚本
```sql
CREATE DATABASE mybatis;

USE mybatis;

CREATE TABLE IF NOT EXISTS `trans`
(
    `id`          BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    `trans_id`    VARCHAR(32)         NOT NULL DEFAULT '' COMMENT '交易单号,唯一',
    `body`        VARCHAR(128)        NOT NULL DEFAULT '' COMMENT '描述',
    `subject`     VARCHAR(128)        NOT NULL DEFAULT '' COMMENT '标题',
    `amount`      BIGINT(20)          NOT NULL DEFAULT 0 COMMENT '交易金额,单位:分',
    `create_time` DATETIME            NOT NULL DEFAULT '1970-01-01 00:00:00' COMMENT '创建时间',
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8 COMMENT ='交易信息表';

INSERT INTO `trans`(`trans_id`, `body`, `subject`, `amount`)
VALUES ('10011000', '微信交易', '支付', 512),
       ('10011001', '支付宝交易', '支付', 1024),
       ('10011002', '易宝支付交易', '支付', 2048),
       ('10011003', '微信交易', '支付', 4096);
```

### 2.ORM
* 实体类:mybatis.domain.Trans
```java
public class Trans {

  private Long id;
  private String trans_id;
  private String body;
  private String subject;
  private Long amount;
  private Date create_time;
}
```

* mybatis.mapper.TransMapper接口
```java
public interface TransMapper {

  // 插入
  int insertTrans(Trans trans);

  // 并返回主键id
  int insertTransAndIdReturn(Trans trans);

  // 修改
  int updateTrans(Trans trans);

  // 删除
  int deleteTransById(Long id);

  // 查询一条记录
  Trans selectOne(String transId);

  // 基于body的模糊查询
  List<Trans> selectByBody(String body);

  // 
  List<Trans> selectByFixedBody(String body);

  // 查询基于查询对象,验证OGNL表达式
  List<Trans> selectByQueryVO(QueryTransVO queryTransVO);

  // 获取全部数据
  int getTotal();
}
```

### 3.建立MyBatis全局配置文件SqlMapConfig.xml
* 位置: resources/SqlMapConfig.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">

<!-- 配置的根节点 -->
<configuration>
  
  <!-- 环境配置 -->
  <environments default="dev">

    <!-- 开发环境:可以配置多个environment节点,用于维护多套环境 -->
    <environment id="dev">

      <!-- 配置事务的类型 -->
      <transactionManager type="JDBC">
      </transactionManager>

      <!-- 配置数据源,取值有三个,POOLED:池 -->
      <dataSource type="POOLED">
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatis?useUnicode=true&amp;characterEncoding=utf-8&amp;zeroDateTimeBehavior=convertToNull"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
      </dataSource>
    </environment>
  </environments>

  <!-- 指定映射的配置文件所在的路径 -->
  <mappers>
    <mapper resource="mybatis/mapper/TransMapper.xml"/>
  </mappers>
</configuration>
```

### 4.在resources下新建Mapper.xml文件
* 位置: resources/mybatis/mapper/TransMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- namespace要与对应接口的全限定名保持一致 -->
<mapper namespace="mybatis.mapper.TransMapper">

  <insert id="insertTrans" parameterType="mybatis.domain.Trans">
    INSERT INTO `trans`(`trans_id`, `body`, `subject`, `amount`, `create_time`) VALUES (#{trans_id}, #{body}, #{subject}, #{amount}, #{create_time})
  </insert>

  <insert id="insertTransAndIdReturn" parameterType="mybatis.domain.Trans">
    <!--
     keyProperty: 实体中定义的属性名
     keyColumn: 数据库表中定义的字段名
     order: AFTER表示插入后的行为
     resultType: 返回值类型
     -->
    <selectKey keyProperty="id" keyColumn="id" order="AFTER" resultType="long">
      SELECT last_insert_id();
    </selectKey>

    INSERT INTO `trans`(`trans_id`, `body`, `subject`, `amount`, `create_time`) VALUES (#{trans_id}, #{body}, #{subject}, #{amount}, #{create_time})
  </insert>

  <update id="updateTrans" parameterType="mybatis.domain.Trans">
    UPDATE `trans` SET `body`=#{body},`subject`=#{subject},`amount`=#{amount} WHERE `trans_id` = #{trans_id}
  </update>

  <delete id="deleteTransById" parameterType="java.lang.Long">
    DELETE FROM `trans` WHERE `id` = #{id}
  </delete>

  <select id="selectOne" parameterType="java.lang.String" resultType="mybatis.domain.Trans">
    SELECT * FROM `trans` WHERE `trans_id` = #{trans_id}
  </select>

  <select id="selectByBody" parameterType="java.lang.String" resultType="mybatis.domain.Trans">
    <!--  Preparing: select * from `trans` where `body` like ? -->
    <!--  mybatis.mapper.TransMapper.selectByBody - ==> Parameters: %支付%(String) -->
    SELECT * FROM `trans` WHERE `body` LIKE #{body}
  </select>

  <select id="selectByFixedBody" parameterType="java.lang.String" resultType="mybatis.domain.Trans">
    <!-- Preparing: select * from `trans` where `body` like '%支付%'  -->
    SELECT * FROM `trans` WHERE `body` LIKE '%${value}%'
  </select>

  <select id="selectByQueryVO" parameterType="mybatis.domain.QueryTransVO" resultType="mybatis.domain.Trans">
    <!-- 基于ognl表达式取出数据
            queryTransVO.trans.body
            queryTransVO.trans.subject
    -->
    SELECT * FROM `trans` WHERE `body` = #{trans.body} AND `subject` = #{trans.subject}
  </select>

  <select id="getTotal" resultType="java.lang.Integer">
    SELECT COUNT(*) FROM `trans`
  </select>
</mapper>
```

### 5.Main程序
```java
package mybatis;

import java.io.IOException;
import java.io.InputStream;
import java.util.Date;
import java.util.List;
import mybatis.domain.QueryTransVO;
import mybatis.domain.Trans;
import mybatis.mapper.TransMapper;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.junit.After;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;

/**
 * @author haoc
 */
public class App {

  private InputStream inputStream;

  private SqlSession sqlSession;

  private TransMapper transMapper;

  /**
   * 插入测试
   */
  @Test
  public void testInsert() {
    Trans trans = new Trans();
    trans.setTrans_id(System.currentTimeMillis() + "");
    trans.setAmount(1024L);
    trans.setBody("其他交易");
    trans.setSubject("支付");
    trans.setCreate_time(new Date());

    int effectRows = transMapper.insertTrans(trans);

    sqlSession.commit();

    Assert.assertEquals(1, effectRows);
  }

  /**
   * 插入并返回id测试
   */
  @Test
  public void testInsertAndIdReturn() {
    Trans trans = new Trans();
    trans.setTrans_id(System.currentTimeMillis() + "");
    trans.setAmount(1024L);
    trans.setBody("其他交易-id");
    trans.setSubject("支付-id");
    trans.setCreate_time(new Date());

    System.out.println("before insert:" + trans);

    int effectRows = transMapper.insertTransAndIdReturn(trans);

    System.out.println("after insert:" + trans);

    sqlSession.commit();

    System.out.println("insert trans ,and effect rows =" + effectRows);
  }

  /**
   * 修改测试
   */
  @Test
  public void testUpdate() {
    Trans trans = new Trans();
    trans.setTrans_id("1558017429079");
    trans.setAmount(2048L);
    trans.setBody("其他交易");
    trans.setSubject("支付");

    int effectRows = transMapper.updateTrans(trans);

    sqlSession.commit();

    Assert.assertEquals(1, effectRows);
  }

  /**
   * 删除测试
   */
  @Test
  public void testDelete() {
    int effectRows = transMapper.deleteTransById(7L);

    sqlSession.commit();

    Assert.assertEquals(1, effectRows);
  }

  /**
   * 查询一条记录
   */
  @Test
  public void testSelectOne() {
    Trans trans = transMapper.selectOne("1558017429079");

    System.out.println(trans);
  }

  /**
   * 查询多条
   */
  @Test
  public void testSelectByBody() {
    // #{body}
    List<Trans> trans = transMapper.selectByBody("%支付%");
    trans.forEach(System.out::println);
  }

  /**
   * 查询多条
   */
  @Test
  public void testSelectByFixedBody() {
    // #{body}
    List<Trans> trans = transMapper.selectByFixedBody("支付");
    trans.forEach(System.out::println);
  }

  /**
   * 查询条包装
   */
  @Test
  public void testSelectByQueryVO() {
    Trans trans = new Trans();
    trans.setBody("其他交易");
    trans.setSubject("支付");

    QueryTransVO queryTransVO = new QueryTransVO();
    queryTransVO.setTrans(trans);

    List<Trans> transes = transMapper.selectByQueryVO(queryTransVO);
    transes.forEach(System.out::println);
  }

  /**
   * 查询全部
   */
  @Test
  public void testGetTotal() {
    int total = transMapper.getTotal();
    System.out.println("total: " + total);
  }

  @Before
  public void init() throws IOException {
    inputStream = Resources.getResourceAsStream("SqlMapConfig.xml");

    // 写法1
    // SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    // sqlSession = sessionFactory.openSession();

    // 写法2
    sqlSession = new SqlSessionFactoryBuilder().build(inputStream).openSession();

    transMapper = sqlSession.getMapper(TransMapper.class);
  }

  @After
  public void destory() throws IOException {
    sqlSession.close();
    inputStream.close();
  }

}
```