---
title: spring装配
date: 2019-01-21 17:05:13
tags: 
- spring
- java
categories:
- java
---


## Spring装配Bean

------

Spring容器配置的三种主要的装配机制：

> * 在XML中进行显示配置
> * 在Java中进行显式配置
> * 隐式的bean发现机制和自动装配

### 1-自动装配
Spring从两个角度实现自动化装配：
> * 组件扫描（component scanning） 自动发现应用上下文中所创建的bean
> * 自动装配（autowiring） Spring自动满足bean之间的依赖


  使用@Component注解来告诉Spring为这个类创建bean
	
	@Component
	public class CDPlayer implements MediaPlayer {
	  private CompactDisc cd;
	
	  @Autowired
	  public CDPlayer(CompactDisc cd) {
	    this.cd = cd;
	  }
	
	  public void play() {
	    cd.play();
	  }
	
	}

通过设置组件扫描可以不用显示配置bean。但是组件扫描默认是不启用的。所以要通过配置来启用。

* 在java配置类中开启组件扫描@ComponentScan

		@Configuration
		@ComponentScan
		public class CDPlayerConfig { 
		}

因为@ComponentScan会扫描与配置类相同的包。可以通过下面两个配置来设置扫描的包：

>  *  @ComponentScan(basePackages={"soundsystem,"sdd"})
> > 扫描包的名字

>  * @ComponentScan(basePackageClasses={CDplayer.class,DVDPlayer.class})
>  > 扫描类所在的包

* 在XML中开启组件扫描
    
		<context:component-scan base-package="soundsystem" />

	
* 为组件扫描的bean命名
> * 通过@Component注解

		
	@Component("sgtPerr")
	public class SgtPeppers implements CompactDisc {
	
	  private String title = "Sgt. Pepper's Lonely Hearts Club Band";  
	  private String artist = "The Beatles";
	  
	  public void play() {
	    System.out.println("Playing " + title + " by " + artist);
	  }
	  
	}
> * 通过@Named注解
	
	@Named("sgtPerr")
	public class SgtPeppers implements CompactDisc {
	
	}


### 2- JAVA代码装配Bean

通过配置类来实现装配Bean


	@Configuration //表明这是一个配置类
	public class CDPlayerConfig {
	  
	  @Bean //声明这是一个bean
	  public CompactDisc compactDisc() {
	    return new SgtPeppers();
	  }
	
	  @Bean
		//通过构造器注入
	  public CDPlayer cdPlayer(CompactDisc compactDisc) {
	    return new CDPlayer(compactDisc);
	  }

	  @Bean 
		//通过setter方法注入
	  public CDPlayer cdPlayer(CompactDisc compactDisc) {
	    CDPlayer cdPlayer = new CDPlayer(compactDisc);
	    cdPlayer.setCompactDisc(compactDisc);
	    return cdPlayer
	  }
	
	}

### 3- 通过XML装配bean

> * 通过构造器注入

	<bean id="compactDisc" class="soundsystem.collections.BlankDisc">
	    <constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
	    <constructor-arg value="The Beatles" />
	    <constructor-arg>
	      <list>
	        <value>Sgt. Pepper's Lonely Hearts Club Band</value>
	        <value>With a Little Help from My Friends</value>
	        <value>Lucy in the Sky with Diamonds</value>
	        <value>Getting Better</value>
	        <value>Fixing a Hole</value>
	        <value>She's Leaving Home</value>
	        <value>Being for the Benefit of Mr. Kite!</value>
	        <value>Within You Without You</value>
	        <value>When I'm Sixty-Four</value>
	        <value>Lovely Rita</value>
	        <value>Good Morning Good Morning</value>
	        <value>Sgt. Pepper's Lonely Hearts Club Band (Reprise)</value>
	        <value>A Day in the Life</value>
	      </list>
	    </constructor-arg>
  	</bean>

> * 通过setter注入

	 <bean id="compactDisc"
        class="soundsystem.properties.BlankDisc">
	    <property name="title" value="Sgt. Pepper's Lonely Hearts Club Band" />
	    <property name="artist" value="The Beatles" />
	    <property name="tracks">
	      <list>
	        <value>Sgt. Pepper's Lonely Hearts Club Band</value>
	        <value>With a Little Help from My Friends</value>
	        <value>Lucy in the Sky with Diamonds</value>
	        <value>Getting Better</value>
	        <value>Fixing a Hole</value>
	        <value>She's Leaving Home</value>
	        <value>Being for the Benefit of Mr. Kite!</value>
	        <value>Within You Without You</value>
	        <value>When I'm Sixty-Four</value>
	        <value>Lovely Rita</value>
	        <value>Good Morning Good Morning</value>
	        <value>Sgt. Pepper's Lonely Hearts Club Band (Reprise)</value>
	        <value>A Day in the Life</value>
	      </list>
	    </property>
  	</bean>


### 4-混合配置
* 在javaConfig引用xml配置

		@Configuration
		@Import(CDPlayerConfig.class) //导入其他的java配置
		@ImportResource("classpath:cd-config.xml") // 导入xml配置
		public class SoundSystemConfig {
		
		}

* 在xml配置中引用javaconfig

		<bean class="soundsystem.CDConfig" /> 导入config配置类
		<import resource="cd-player-config.xml"/> 导入xml配置

---
##高级装配

### 1- 环境与profile
