---
title: solr简单使用
date: 2019-04-02 20:36:24
tags: 
- solr
- spring-data-solr
categories:
- solr
- spring
---
# solr的安装与配置

----------

##  solr简介 


　　Solr（读作“solar”）是Apache Lucene项目的开源企业搜索平台。其主要功能包括全文检索、命中标示[1]、分面搜索、动态聚类、数据库集成，以及富文本（如Word、PDF）的处理。Solr是高度可扩展的，并提供了分布式搜索和索引复制。Solr是最流行的企业级搜索引擎，[2]Solr 4还增加了NoSQL支持。[3]

　　Solr是用Java编写、运行在Servlet容器（如Apache Tomcat或Jetty）的一个独立的全文搜索服务器。 Solr采用了Lucene Java搜索库为核心的全文索引和搜索，并具有类似REST的HTTP/XML和JSON的API。 Solr强大的外部配置功能使得无需进行Java编码，便可对其进行调整以适应多种类型的应用程序。Solr有一个插件架构，以支持更多的高级定制。

##  solr的安装

1. 下载solr 8.0版并解压 ，下载地址为： [https://www-eu.apache.org/dist/lucene/solr/8.0.0/](https://www-eu.apache.org/dist/lucene/solr/8.0.0/ "下载地址")
2. 下载并安装tomcat
3. 将解压后solr-8.0.0\server\solr-webapp\webapp这个文件夹复制到tomcat的apache-tomcat-9.0.14\webapps文件夹下，并改名为solr（为了方便访问）
4. 把solr下example/lib/ext 目录下的所有的 jar 包，添加到 solr 的工程中(\WEB-INF\lib目录下)。
5. 在任意位置创建solr-home 目录如D:\solrhome，然后将solr-8.0.0\server\solr文件夹复制到该目录下
6. 创建collection：在solr目录下创建文件夹firstcore（也就是集合），将\solr\configsets\sample_techproducts_configs下的conf文件复制到该目录下
7. 关联 solr 及 solrhome。需要修改 solr 工程的 web.xml 文件。
	
	    <env-entry>
	       <env-entry-name>solr/home</env-entry-name>
	       <env-entry-value>d:\solrhome</env-entry-value>
	       <env-entry-type>java.lang.String</env-entry-type>
	    </env-entry>

8. 启动tomcat   apache-tomcat-9.0.14\bin\startup.bat

##  配置中文分析器IK Analyzer

### 下载IK Analyzer：
 
　　　因为solr为8.0版本，网上常见的IK Analyzer版本为IKAnalyzer2012FF_u1.jar无法使用，需要下载最新版本，下载网址为：[https://github.com/magese/ik-analyzer-solr7](https://github.com/magese/ik-analyzer-solr7 "IK")
 

###  在solr中配置IK Analyzer

1. 将resources目录下的5个配置文件放入的solr的服务Jetty或Tomcat的webapp/solr/WEB-INF/classes/目录下;

		①IKAnalyzer.cfg.xml 
		②ext.dic 
		③stopword.dic 
		④ik.conf 
		⑤dynamicdic.txt
2. 配置的Solr的managed-schema(路径为solr\firstCore\conf)，添加ik分词器，示例如下;

		<！ -  ik分词器 - > 
		<fieldType name =“text_ik”class =“solr.TextField”> 
		  <analyzer type =“index”> 
		      <tokenizer class =“org.wltea.analyzer.lucene.IKTokenizerFactory”useSmart = “false”conf =“ik.conf”/> 
		      <filter class =“solr.LowerCaseFilterFactory”/> 
		  </ analyzer> 
		  <analyzer type =“query”> 
		      <tokenizer class =“org.wltea.analyzer.lucene.IKTokenizerFactory” useSmart =“true”conf =“ik.conf”/> 
		      <filter class =“solr.LowerCaseFilterFactory”/> 
		  </ analyzer> 
		</ fieldType>

##  配置域
　　域相当于数据库的表字段，用户存放数据，因此用户根据业务需要去定义相关的Field（域），一般来说，每一种对应着一种数据，用户对同一种数据进行相同的操作。

　　域的常用属性：

	•	name：指定域的名称
	•	type：指定域的类型
	•	indexed：是否索引
	•	stored：是否存储
	•	required：是否必须
	•	multiValued：是否多值

### 域
修改solrhome的schema.xml文件,添加field

	<field name="item_title" type="text_ik" indexed="true" stored="true"/>
	

### 复制域

复制域的作用在于将某一个Field中的数据复制到另一个域中(可以用于多字段搜索)

	<field name="item_keywords" type="text_ik" indexed="true" stored="false" multiValued="true"/>
	<copyField source="item_title" dest="item_keywords"/>
	<copyField source="item_name" dest="item_keywords"/>


### 动态域
当我们需要动态扩充字段时，我们需要使用动态域。需要实现的效果如下：

	<dynamicField name="item_spec_*" type="string" indexed="true" stored="true" />	



#  Spring Data Solr的简单使用


## 入门demo

1. 创建maven工程，pom.xml中引入依赖
		
		<dependencies>
			<dependency>
			    <groupId>org.springframework.data</groupId>
			    <artifactId>spring-data-solr</artifactId>
			    <version>1.5.5.RELEASE</version>
			</dependency> 
			<dependency>
				<groupId>org.springframework</groupId>
				<artifactId>spring-test</artifactId>
				<version>4.2.4.RELEASE</version>
			</dependency>
			<dependency>
				<groupId>junit</groupId>
				<artifactId>junit</artifactId>
				<version>4.9</version>
			</dependency>
		</dependencies>

2. 在src/main/resources下创建  applicationContext-solr.xml

		<?xml version="1.0" encoding="UTF-8"?>
		<beans xmlns="http://www.springframework.org/schema/beans"
			xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
			xmlns:context="http://www.springframework.org/schema/context"
			xmlns:solr="http://www.springframework.org/schema/data/solr"
			xsi:schemaLocation="http://www.springframework.org/schema/data/solr 
		  		http://www.springframework.org/schema/data/solr/spring-solr-1.0.xsd
				http://www.springframework.org/schema/beans 
				http://www.springframework.org/schema/beans/spring-beans.xsd
				http://www.springframework.org/schema/context 
				http://www.springframework.org/schema/context/spring-context.xsd">
			<!-- solr服务器地址,记得加上集合名 -->
			<solr:solr-server id="solrServer" url="http://127.0.0.1:8080/solr/firstcore" />
			<!-- solr模板，使用solr模板可对索引库进行CRUD的操作 -->
			<bean id="solrTemplate" class="org.springframework.data.solr.core.SolrTemplate">
				<constructor-arg ref="solrServer" />
			</bean>
		</beans>
    
3. 将需要导入solr的实体类添加注解@Field

     如果属性与配置文件定义的域名称不一致，需要在注解中指定域名称。如同下面的title，在solr的配置文件中名为item_title, 所以要加上注解`@Field("item_titel")`,其他也一样。（）


		public class TbItem implements Serializable{
	
			@Field
		    private Long id;
		
			@Field("item_title")
		    private String title;
			    
		    @Field("item_price")
			private BigDecimal price;
		
		   
		}

4. 创建测试类TestTemplate.java

		@RunWith(SpringJUnit4ClassRunner.class)
		@ContextConfiguration(locations="classpath:applicationContext-solr.xml")
		public class TestTemplate {
		
			@Autowired
			private SolrTemplate solrTemplate;

			//增加到solr索引库
			@Test
			public void testAdd(){
				TbItem item=new TbItem();
				item.setId(1L);
				item.setTitle("barktegh");
				item.setPrice(new BigDecimal(2000));		
				solrTemplate.saveBean(item);
				solrTemplate.commit();
			}

			//按主键查询
			@Test
			public void testFindOne(){
				TbItem item = solrTemplate.getById(1, TbItem.class);
				System.out.println(item.getTitle());
			}

			//循环项solr插入数据
			@Test
			public void testAddList(){
				List<TbItem> list=new ArrayList();
				
				for(int i=0;i<100;i++){
					TbItem item=new TbItem();
					item.setId(i+1L);
					item.setTitle("华为Mate"+i);
					item.setPrice(new BigDecimal(2000+i));	
					list.add(item);
				}
				
				solrTemplate.saveBeans(list);
				solrTemplate.commit();
			}

			//分页查询
			@Test
			public void testPageQuery(){
				Query query=new SimpleQuery("*:*");
				query.setOffset(20);//开始索引（默认0）
				query.setRows(20);//每页记录数(默认10)
				ScoredPage<TbItem> page = solrTemplate.queryForPage(query, TbItem.class);
				System.out.println("总记录数："+page.getTotalElements());
				List<TbItem> list = page.getContent();
				showList(list);
			}	



			//条件查询
			@Test
			public void testPageQueryMutil(){	
				Query query=new SimpleQuery("*:*");
				Criteria criteria=new Criteria("item_title").contains("2");
				criteria=criteria.and("item_title").contains("5");		
				query.addCriteria(criteria);
				//query.setOffset(20);//开始索引（默认0）
				//query.setRows(20);//每页记录数(默认10)
				ScoredPage<TbItem> page = solrTemplate.queryForPage(query, TbItem.class);
				System.out.println("总记录数："+page.getTotalElements());
				List<TbItem> list = page.getContent();
				showList(list);
			}
			
			//删除全部数据
			@Test
			public void testDeleteAll(){
				Query query=new SimpleQuery("*:*");
				solrTemplate.delete(query);
				solrTemplate.commit();
			}


			//显示记录数据
			private void showList(List<TbItem> list){		
				for(TbItem item:list){
					System.out.println(item.getTitle() +item.getPrice());
				}		
			}

		}

5. 在 http://127.0.0.1:8080/solr 可以查询到数据变化

		





