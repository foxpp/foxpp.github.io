---
layout:     post
title:      "Activiti-2-Hello-world"
subtitle:   "Activiti-2-Hello-world"
date:       2016-1-11
author:     "fox"
header-img: "img/post-bg-01.jpg"
---



#Activiti-2-Hello-world

---


## 快速开始一个helloworld

在activiti中
有4钟方法来声明java调用逻辑：

*	实现JavaDelegate或ActivityBehavior
*	执行解析代理对象的表达式
*	调用一个方法表达式
*	调用一直值表达式

在包com.demo.jclass下新建javaclass: TestDelClass ,实现JavaDelegate，ide会提示你实现方法

~~~java
package com.demo.jclass;

import org.activiti.engine.delegate.DelegateExecution;
import org.activiti.engine.delegate.JavaDelegate;


public class TestDelClass implements JavaDelegate {
    public void execute(DelegateExecution execution) throws Exception {
        System.out.println("Hi activiti! ");
    }
}

~~~





>或许有人会问JavaDelegate或ActivityBehavior有什么区别，区别就是JavaDelegate提供的功能没有ActivityBehavior的复杂和强大。
实现ActivityBehavior的类可以影响流程的走向，如果不是非常复杂，就用JavaDelegate就够了




接下来是工作流的定义文件，由于是一开始，我们就用最简单的方式来写，用activiti:class指定类就可以，activiti:delegateExpression，activiti:expression这样的语法适用于指定spring的bean.

classpath下新建流程文件 java\_class\_demo.bpmn20.xml,指定刚才创建的类到serviceTask上面,代码如下

~~~xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<definitions id="definitions"
             targetNamespace="http://activiti.org/bpmn20"
             xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             xmlns:activiti="http://activiti.org/bpmn">

    <process id="java_class_demo">
        <startEvent id="demoStart" name="start"/>
        <sequenceFlow sourceRef="demoStart" targetRef="javaSeviceTask"/>
        <serviceTask id="javaSeviceTask" name="sayHello" activiti:class="com.demo.jclass.TestDelClass"/>

        <sequenceFlow id="flow" sourceRef="javaSeviceTask" targetRef="end"/>
        <endEvent id="end" name="End"/>
    </process>
</definitions>
~~~



然后随便建立另一个类，按照创建引擎->部署流程->启动指定工作流的顺序来写，代码如下

~~~java 
package com.demo.process;

import org.activiti.engine.ProcessEngine;
import org.activiti.engine.ProcessEngineConfiguration;
import org.activiti.engine.RepositoryService;
import org.activiti.engine.RuntimeService;
import org.activiti.engine.repository.ProcessDefinition;

public class SimpleProcessDemo {


    public static void main(String[] args) {
        ProcessEngine processEngine =  ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration().buildProcessEngine();
        RepositoryService repositoryService = processEngine.getRepositoryService();
        repositoryService.createDeployment().addClasspathResource("workflow/java_class_demo.bpmn20.xml").deploy();
        ProcessDefinition processDefinition = repositoryService.createProcessDefinitionQuery().singleResult();
        System.out.println(processDefinition.getKey());

        RuntimeService runtimeService = processEngine.getRuntimeService();
        runtimeService.startProcessInstanceByKey("java_class_demo");
    }
}

~~~



若一切顺利，你会看到这样的输出：

~~~
java_class_demo
hello , i'm from TestClass

Process finished with exit code 0
~~~





##使用流程中的变量

activit工作流task之间自始至终携带着一个map的变量集合，可以在流程一开始的时候便设置一些值进去，在过程中也可以添加删除或者改变。


我们来设计一个简单的携带变量的流程，在流程开始时，向javaSeviceTask1写入一些变量，然后在里面打印出来，在javaSeviceTask1流向javaSeviceTask2之前，再写入一些变量，在javaSeviceTask2中打印从javaSeviceTask1中获取的变量

修改或者新建classpath下的java\_class\_demo.bpmn20.xml,代码如下

~~~xml

<?xml version="1.0" encoding="ISO-8859-1"?>
<definitions id="definitions"
             targetNamespace="http://activiti.org/bpmn20"
             xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             xmlns:activiti="http://activiti.org/bpmn">

    <process id="java_class_demo">
        <startEvent id="demoStart" name="start"/>
        <sequenceFlow sourceRef="demoStart" targetRef="javaSeviceTask1"/>
        <serviceTask id="javaSeviceTask1" name="useVar1" activiti:class="com.demo.jclass.UseVarClsA"/>
        <sequenceFlow id="flow1" sourceRef="javaSeviceTask1" targetRef="javaServiceTask2"/>
        <serviceTask id="javaServiceTask2" name="useVar2" activiti:class="com.demo.jclass.UseVarClsB"/>
        <sequenceFlow sourceRef="javaServiceTask2" targetRef="end"/>
        <endEvent id="end" name="End"/>
    </process>
</definitions>
~~~

修改SimpleProcessDemo类的代码,写入一些变量,启动流程的时候附带进流程中，如下


~~~java
package com.demo.process;

import org.activiti.engine.ProcessEngine;
import org.activiti.engine.ProcessEngineConfiguration;
import org.activiti.engine.RepositoryService;
import org.activiti.engine.RuntimeService;
import org.activiti.engine.repository.ProcessDefinition;

import java.util.HashMap;
import java.util.Map;

public class SimpleProcessDemo {


    public static void main(String[] args) {
        ProcessEngine processEngine =  ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration().buildProcessEngine();
        RepositoryService repositoryService = processEngine.getRepositoryService();
        repositoryService.createDeployment().addClasspathResource("workflow/java_class_demo.bpmn20.xml").deploy();
        ProcessDefinition processDefinition = repositoryService.createProcessDefinitionQuery().singleResult();
        System.out.println(processDefinition.getKey());

        RuntimeService runtimeService = processEngine.getRuntimeService();
        Map<String,Object> paras = new HashMap<String, Object>();
        paras.put("Jack","15");
        paras.put("Tom","16");
        runtimeService.startProcessInstanceByKey("java_class_demo", paras);
    }
}

~~~


然后com.demo.jclass包下新建class, UseVarClsA,在其中打印一开始设置进来的值，然后写进去一个随机值，key是"someVarFromUseVarClsA",如下

~~~java
package com.demo.jclass;

import org.activiti.engine.delegate.DelegateExecution;
import org.activiti.engine.delegate.JavaDelegate;

import java.util.Map;
import java.util.Random;

public class UseVarClsA implements JavaDelegate {
    public void execute(DelegateExecution execution) throws Exception {
        Map<String,Object> paras = execution.getVariables();
        for (Map.Entry<String, Object> stringObjectEntry : paras.entrySet()) {
            System.out.println(stringObjectEntry.getKey() + ":"+ stringObjectEntry.getValue());
        }
        execution.setVariable("someVarFromUseVarClsA",new Random().nextInt());

    }
}

~~~


然后com.demo.jclass包下新建class, UseVarClsB,打印从上一个节点带过来的值:即"someVarFromUseVarClsA",代码如下

~~~java
package com.demo.jclass;

import org.activiti.engine.delegate.DelegateExecution;
import org.activiti.engine.delegate.JavaDelegate;

public class UseVarClsB implements JavaDelegate {
    public void execute(DelegateExecution execution) throws Exception {
        Integer fromA = Integer.parseInt(execution.getVariable("someVarFromUseVarClsA").toString());
        System.out.println("I get var from A :" + fromA);
    }
}
~~~

一切顺利的话，控制台打印将会是下面的话

~~~
java_class_demo
Jack:15
Tom:16
I get var from A :-275413182
~~~


