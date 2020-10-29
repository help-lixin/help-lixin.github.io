---
layout: post
title: 'Activiti源码(BpmnXMLConverter)'
date: 2020-01-24
author: 李新
tags: Activiti
---

### (1).创建 BpmnXMLConverter
```
// 资源文件
InputStream is = UserTaskTest.class.getClassLoader().getResource("user-task.bpmn").openStream();
// 对资源文件进行wrapper
InputStreamSource inputStreamProvider = new InputStreamSource(is);
// 创建BpmnXMLConverter对象
BpmnXMLConverter converter = new BpmnXMLConverter();
// 解析InputStream转换成:BpmnModel
BpmnModel model = converter.convertToBpmnModel(inputStreamProvider, true, false, "UTF-8");
```
### (2).BpmnXMLConverter 源码
```
public class BpmnXMLConverter implements BpmnXMLConstants {
    // field
    // 实例化一系列元素解析器
    
    // key   : xml中的节点,比如:startEvent
    // value : 对应的处理器
    protected static Map<String, BaseBpmnXMLConverter> convertersToBpmnMap = new HashMap<String, BaseBpmnXMLConverter>(); 
    
    // key   : 解析xml节点之后,对应的载体(实体)
   // value :
    protected static Map<Class<? extends BaseElement>, BaseBpmnXMLConverter> convertersToXMLMap = new HashMap<Class<? extends BaseElement>, BaseBpmnXMLConverter>();

    //元素解析器添加过程
    static {
    // events
    addConverter(new EndEventXMLConverter());
    addConverter(new StartEventXMLConverter());

    // tasks
    addConverter(new BusinessRuleTaskXMLConverter());
    addConverter(new ManualTaskXMLConverter());
    addConverter(new ReceiveTaskXMLConverter());
    addConverter(new ScriptTaskXMLConverter());
    addConverter(new ServiceTaskXMLConverter());
    addConverter(new SendTaskXMLConverter());
    addConverter(new UserTaskXMLConverter());
    addConverter(new TaskXMLConverter());
    addConverter(new CallActivityXMLConverter());

    // gateways
    addConverter(new EventGatewayXMLConverter());
    addConverter(new ExclusiveGatewayXMLConverter());
    addConverter(new InclusiveGatewayXMLConverter());
    addConverter(new ParallelGatewayXMLConverter());
    addConverter(new ComplexGatewayXMLConverter());

    // connectors
    addConverter(new SequenceFlowXMLConverter());

    // catch, throw and boundary event
    addConverter(new CatchEventXMLConverter());
    addConverter(new ThrowEventXMLConverter());
    addConverter(new BoundaryEventXMLConverter());

    // artifacts
    addConverter(new TextAnnotationXMLConverter());
    addConverter(new AssociationXMLConverter());

    // data store reference
    addConverter(new DataStoreReferenceXMLConverter());


    // data objects
    // ValuedDataObjectXMLConverter 负责解析dataObject元素
    addConverter(new ValuedDataObjectXMLConverter(), StringDataObject.class);
    addConverter(new ValuedDataObjectXMLConverter(), BooleanDataObject.class);
    addConverter(new ValuedDataObjectXMLConverter(), IntegerDataObject.class);
    addConverter(new ValuedDataObjectXMLConverter(), LongDataObject.class);
    addConverter(new ValuedDataObjectXMLConverter(), DoubleDataObject.class);
    addConverter(new ValuedDataObjectXMLConverter(), DateDataObject.class);

    // Alfresco types
    addConverter(new AlfrescoStartEventXMLConverter());
    addConverter(new AlfrescoUserTaskXMLConverter());
  } //end static

  // 添加转换器
  public static void addConverter(BaseBpmnXMLConverter converter) {
    // converter.getBpmnElementType() == entity
    addConverter(converter, converter.getBpmnElementType());
  }
  
  // 添加转换器
  public static void addConverter(BaseBpmnXMLConverter converter, Class<? extends BaseElement> elementType) {
    // converter.getXMLElementName() == xml节点
    convertersToBpmnMap.put(converter.getXMLElementName(), converter);
    convertersToXMLMap.put(elementType, converter);
  }
  
  
}
```
### (3).BpmnXMLConverter.convertToBpmnModel
```
public BpmnModel convertToBpmnModel(InputStreamProvider inputStreamProvider, boolean validateSchema, boolean enableSafeBpmnXml, String encoding) {
    XMLInputFactory xif = XMLInputFactory.newInstance();
    if (xif.isPropertySupported(XMLInputFactory.IS_REPLACING_ENTITY_REFERENCES)) {
      xif.setProperty(XMLInputFactory.IS_REPLACING_ENTITY_REFERENCES, false);
    }
    if (xif.isPropertySupported(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES)) {
      xif.setProperty(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false);
    }
    if (xif.isPropertySupported(XMLInputFactory.SUPPORT_DTD)) {
      xif.setProperty(XMLInputFactory.SUPPORT_DTD, false);
    }
    InputStreamReader in = null;
    try {
      in = new InputStreamReader(inputStreamProvider.getInputStream(), encoding);
      XMLStreamReader xtr = xif.createXMLStreamReader(in);

      try {
        if (validateSchema) {
          if (!enableSafeBpmnXml) {
            validateModel(inputStreamProvider);
          } else {
            validateModel(xtr);
          }

          // The input stream is closed after schema validation
          in = new InputStreamReader(inputStreamProvider.getInputStream(), encoding);
          xtr = xif.createXMLStreamReader(in);
        } //end if
      } catch (Exception e) {
        throw new XMLException(e.getMessage(), e);
      }
      // ********************************************************************
      // XML conversion
      // 读取XML并转换
      // ********************************************************************
      return convertToBpmnModel(xtr);
    } catch (UnsupportedEncodingException e) {
      throw new XMLException("The bpmn 2.0 xml is not UTF8 encoded", e);
    } catch (XMLStreamException e) {
      throw new XMLException("Error while reading the BPMN 2.0 XML", e);
    } finally {
      if (in != null) {
        try {
          in.close();
        } catch (IOException e) {
          LOGGER.debug("Problem closing BPMN input stream", e);
        }
      }
    }
  }
```
### (4).BpmnXMLConverter.convertToBpmnModel
```
public BpmnModel convertToBpmnModel(XMLStreamReader xtr) {
    // 创建返回后的结果集
    BpmnModel model = new BpmnModel();
    model.setStartEventFormTypes(startEventFormTypes);
    model.setUserTaskFormTypes(userTaskFormTypes);
    try {
      Process activeProcess = null;
      List<SubProcess> activeSubProcessList = new ArrayList<SubProcess>();
      while (xtr.hasNext()) {
        try {
          xtr.next();
        } catch (Exception e) {
          LOGGER.debug("Error reading XML document", e);
          throw new XMLException("Error reading XML", e);
        }

        if (xtr.isEndElement() && (ELEMENT_SUBPROCESS.equals(xtr.getLocalName()) || 
            ELEMENT_TRANSACTION.equals(xtr.getLocalName()) ||
            ELEMENT_ADHOC_SUBPROCESS.equals(xtr.getLocalName()))) {          
          activeSubProcessList.remove(activeSubProcessList.size() - 1);
        }
        
        if (xtr.isStartElement() == false) {
            // 如果不是开始节点,则跳过这个节点
          continue;
        }
        
       if (ELEMENT_DEFINITIONS.equals(xtr.getLocalName())) { // <definitions>
            // **********************************************************
            definitionsParser.parse(xtr, model);		  
            // **********************************************************
        } else if (ELEMENT_RESOURCE.equals(xtr.getLocalName())) { // <resource>
            resourceParser.parse(xtr, model);          
        } else if (ELEMENT_SIGNAL.equals(xtr.getLocalName())) {  // <signal>
            signalParser.parse(xtr, model);			
        } else if (ELEMENT_MESSAGE.equals(xtr.getLocalName())) { // <message>
            messageParser.parse(xtr, model);
        } else if (ELEMENT_ERROR.equals(xtr.getLocalName())) {   // <error>
           if (StringUtils.isNotEmpty(xtr.getAttributeValue(null, ATTRIBUTE_ID))) {
               model.addError(xtr.getAttributeValue(null, ATTRIBUTE_ID), xtr.getAttributeValue(null, ATTRIBUTE_ERROR_CODE));
           }
        } else if (ELEMENT_IMPORT.equals(xtr.getLocalName())) {  // <import>
              importParser.parse(xtr, model);
        } else if (ELEMENT_ITEM_DEFINITION.equals(xtr.getLocalName())) {  // <itemDefinition>
              itemDefinitionParser.parse(xtr, model);
        } else if (ELEMENT_DATA_STORE.equals(xtr.getLocalName())) {   // <dataStore>
              dataStoreParser.parse(xtr, model);
        } else if (ELEMENT_INTERFACE.equals(xtr.getLocalName())) {   // <interface>
              interfaceParser.parse(xtr, model);
        } else if (ELEMENT_IOSPECIFICATION.equals(xtr.getLocalName())) {  // <ioSpecification>
              ioSpecificationParser.parseChildElement(xtr, activeProcess, model);
        } else if (ELEMENT_PARTICIPANT.equals(xtr.getLocalName())) {   // <participant>
              participantParser.parse(xtr, model);
        } else if (ELEMENT_MESSAGE_FLOW.equals(xtr.getLocalName())) {   // <messageFlow>
              messageFlowParser.parse(xtr, model);
        } else if (ELEMENT_PROCESS.equals(xtr.getLocalName())) {   // <process>
              Process process = processParser.parse(xtr, model);
              if (process != null) {
                activeProcess = process;
              }
        } else if (ELEMENT_POTENTIAL_STARTER.equals(xtr.getLocalName())) {  // <potentialStarter>
              potentialStarterParser.parse(xtr, activeProcess);
        } else if (ELEMENT_LANE.equals(xtr.getLocalName())) {   // <lane>
              laneParser.parse(xtr, activeProcess, model);
        } else if (ELEMENT_DOCUMENTATION.equals(xtr.getLocalName())) {  //<documentation>
              BaseElement parentElement = null;
              if (!activeSubProcessList.isEmpty()) {
                parentElement = activeSubProcessList.get(activeSubProcessList.size() - 1);
              } else if (activeProcess != null) {
                parentElement = activeProcess;
              }
              documentationParser.parseChildElement(xtr, parentElement, model);
        } else if (activeProcess == null && ELEMENT_TEXT_ANNOTATION.equals(xtr.getLocalName())) {  // <textAnnotation>
              String elementId = xtr.getAttributeValue(null, ATTRIBUTE_ID);
              TextAnnotation textAnnotation = (TextAnnotation) new TextAnnotationXMLConverter().convertXMLToElement(xtr, model);
              textAnnotation.setId(elementId);
              model.getGlobalArtifacts().add(textAnnotation);
        } else if (activeProcess == null && ELEMENT_ASSOCIATION.equals(xtr.getLocalName())) {  // <association>
              String elementId = xtr.getAttributeValue(null, ATTRIBUTE_ID);
              Association association = (Association) new AssociationXMLConverter().convertXMLToElement(xtr, model);
              association.setId(elementId);
              model.getGlobalArtifacts().add(association);
        } else if (ELEMENT_EXTENSIONS.equals(xtr.getLocalName())) {   // <extensionElements>
              extensionElementsParser.parse(xtr, activeSubProcessList, activeProcess, model);
        } else if (ELEMENT_SUBPROCESS.equals(xtr.getLocalName()) || ELEMENT_TRANSACTION.equals(xtr.getLocalName()) || ELEMENT_ADHOC_SUBPROCESS.equals(xtr.getLocalName())) {  // <subProcess>/transaction
              subProcessParser.parse(xtr, activeSubProcessList, activeProcess);       
        } else if (ELEMENT_COMPLETION_CONDITION.equals(xtr.getLocalName())) {   // <completionCondition>
              if (!activeSubProcessList.isEmpty()) {
                SubProcess subProcess = activeSubProcessList.get(activeSubProcessList.size() - 1);
                if (subProcess instanceof AdhocSubProcess) {
                  AdhocSubProcess adhocSubProcess = (AdhocSubProcess) subProcess;
                  adhocSubProcess.setCompletionCondition(xtr.getElementText());
                } //end if
              } // end if
        } else if (ELEMENT_DI_SHAPE.equals(xtr.getLocalName())) {  //<BPMNShape>
              bpmnShapeParser.parse(xtr, model);
        } else if (ELEMENT_DI_EDGE.equals(xtr.getLocalName())) {  // BPMNEdge
              bpmnEdgeParser.parse(xtr, model);
        } else {
              if (!activeSubProcessList.isEmpty() && ELEMENT_MULTIINSTANCE.equalsIgnoreCase(xtr.getLocalName())) {  //<multiInstanceLoopCharacteristics>
                multiInstanceParser.parseChildElement(xtr, activeSubProcessList.get(activeSubProcessList.size() - 1), model);
              } else if (convertersToBpmnMap.containsKey(xtr.getLocalName())) {   // 预留给业务的处理
                if (activeProcess != null) {
                  //  ****************************************************************
                  BaseBpmnXMLConverter converter = convertersToBpmnMap.get(xtr.getLocalName());
                  converter.convertToBpmnModel(xtr, model, activeProcess, activeSubProcessList);
                }
             } //end else if
        } //end else
      } //end while

      // 处理所有的Processes
      for (Process process : model.getProcesses()) {
        for (Pool pool : model.getPools()) {
          if (process.getId().equals(pool.getProcessRef())) {
            pool.setExecutable(process.isExecutable());
          }
        }
        processFlowElements(process.getFlowElements(), process);
      }
    } catch (XMLException e) {
      throw e;
    } catch (Exception e) {
      LOGGER.error("Error processing BPMN document", e);
      throw new XMLException("Error processing BPMN document", e);
    }
    return model;
}
```
### (5).解析<definitions>
```
public class DefinitionsParser implements BpmnXMLConstants {
  // 默认的属性列表
  protected static final List<ExtensionAttribute> defaultAttributes = 
    Arrays.asList(
        new ExtensionAttribute(TYPE_LANGUAGE_ATTRIBUTE), 
        new ExtensionAttribute(EXPRESSION_LANGUAGE_ATTRIBUTE),
        new ExtensionAttribute(TARGET_NAMESPACE_ATTRIBUTE)
   );

  @SuppressWarnings("unchecked")
  public void parse(XMLStreamReader xtr, BpmnModel model) throws Exception {
    // <definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:activiti="http://activiti.org/bpmn" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC" xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI" typeLanguage="http://www.w3.org/2001/XMLSchema" expressionLanguage="http://www.w3.org/1999/XPath" targetNamespace="http://www.activiti.org/test"></definitions>
    // 解析属性:targetNamespace,并设置到实例上
    model.setTargetNamespace(xtr.getAttributeValue(null, TARGET_NAMESPACE_ATTRIBUTE));
    // 解析namespace的数量
    for (int i = 0; i < xtr.getNamespaceCount(); i++) {
      String prefix = xtr.getNamespacePrefix(i);
      if (StringUtils.isNotEmpty(prefix)) {
        model.addNamespace(prefix, xtr.getNamespaceURI(i));
      }
    }

    // 解析:属性数量
    for (int i = 0; i < xtr.getAttributeCount(); i++) {
      ExtensionAttribute extensionAttribute = new ExtensionAttribute();
      extensionAttribute.setName(xtr.getAttributeLocalName(i));
      extensionAttribute.setValue(xtr.getAttributeValue(i));
      if (StringUtils.isNotEmpty(xtr.getAttributeNamespace(i))) {
        extensionAttribute.setNamespace(xtr.getAttributeNamespace(i));
      }
      if (StringUtils.isNotEmpty(xtr.getAttributePrefix(i))) {
        extensionAttribute.setNamespacePrefix(xtr.getAttributePrefix(i));
      }
      // 属性不在默认列表之内,则添加到实体里
      if (!BpmnXMLUtil.isBlacklisted(extensionAttribute, defaultAttributes)) {
        model.addDefinitionsAttribute(extensionAttribute);
      }
    }
  } //end parse
  
}
```
### (6).解析<process>
```
/**
*  <process id="process1" name="My process" isExecutable="true">
*    ... ...
*  </process>  
*/
public class ProcessParser implements BpmnXMLConstants {
    
    public Process parse(XMLStreamReader xtr, BpmnModel model) throws Exception {
        Process process = null;
        if (StringUtils.isNotEmpty(xtr.getAttributeValue(null, ATTRIBUTE_ID))) {
          // 解析id="process1"
          String processId = xtr.getAttributeValue(null, ATTRIBUTE_ID);
          process = new Process();
          process.setId(processId);
          // XML中位置
          BpmnXMLUtil.addXMLLocation(process, xtr);
          // 解析name="My process"
          process.setName(xtr.getAttributeValue(null, ATTRIBUTE_NAME));
          // 解析isExecutable="true"
          if (StringUtils.isNotEmpty(xtr.getAttributeValue(null, ATTRIBUTE_PROCESS_EXECUTABLE))) {
            process.setExecutable(Boolean.parseBoolean(xtr.getAttributeValue(null, ATTRIBUTE_PROCESS_EXECUTABLE)));
          }
          // 解析candidateStarterUsers="${zhangsan,lishi}"
          String candidateUsersString = xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_PROCESS_CANDIDATE_USERS);
          if (StringUtils.isNotEmpty(candidateUsersString)) {
            //
            List<String> candidateUsers = BpmnXMLUtil.parseDelimitedList(candidateUsersString);
            process.setCandidateStarterUsers(candidateUsers);
          }
          // 解析candidateStarterGroups="${group1}"
          String candidateGroupsString = xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_PROCESS_CANDIDATE_GROUPS);
          if (StringUtils.isNotEmpty(candidateGroupsString)) {
            List<String> candidateGroups = BpmnXMLUtil.parseDelimitedList(candidateGroupsString);
            process.setCandidateStarterGroups(candidateGroups);
          }
          // 解析自定义的属性 
          BpmnXMLUtil.addCustomAttributes(xtr, process, ProcessExport.defaultProcessAttributes);
          // 添加解析的process到实体中
          // 证明一个模型中允许存中多个<process>
          model.getProcesses().add(process);
        }
        return process;
  } //end parse
    
}
```
### (7).解析<userTask>
```
/**
*  <userTask id="usertask1" name="任务节点1" activiti:assignee="lixin" activiti:author="Hello World!!!">
*  </userTask>
*/
public abstract class BaseBpmnXMLConverter implements BpmnXMLConstants {
  protected static final List<ExtensionAttribute> defaultElementAttributes = 
     Arrays.asList(
         new ExtensionAttribute(ATTRIBUTE_ID), 
         new ExtensionAttribute(ATTRIBUTE_NAME)
     );

  protected static final List<ExtensionAttribute> defaultActivityAttributes = 
    Arrays.asList(
        new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_ACTIVITY_ASYNCHRONOUS),
        new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_ACTIVITY_EXCLUSIVE), 
        new ExtensionAttribute(ATTRIBUTE_DEFAULT), 
        new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE,ATTRIBUTE_ACTIVITY_ISFORCOMPENSATION)
  );
  
  public void convertToBpmnModel(XMLStreamReader xtr, BpmnModel model, Process activeProcess, List<SubProcess> activeSubProcessList) throws Exception {
      
    // 解析:id
    String elementId = xtr.getAttributeValue(null, ATTRIBUTE_ID);
    // 解析:name
    String elementName = xtr.getAttributeValue(null, ATTRIBUTE_NAME);
    // 解析:async
    boolean async = parseAsync(xtr);
    // 解析:exclusive
    boolean notExclusive = parseNotExclusive(xtr);
    // 解析:default
    String defaultFlow = xtr.getAttributeValue(null, ATTRIBUTE_DEFAULT);
    // 解析:isForCompensation
    boolean isForCompensation = parseForCompensation(xtr);
    // 交给子类去实现(UserTaskXMLConverter)
    BaseElement parsedElement = convertXMLToElement(xtr, model);
    
    // 当节点是:Artifact的处理
    if (parsedElement instanceof Artifact) {
      Artifact currentArtifact = (Artifact) parsedElement;
      currentArtifact.setId(elementId);
      if (!activeSubProcessList.isEmpty()) {
        activeSubProcessList.get(activeSubProcessList.size() - 1).addArtifact(currentArtifact);

      } else {
        activeProcess.addArtifact(currentArtifact);
      }
    }

    // 当节点是:FlowElement处理(userTask继承于:FlowElement)
    // 证明: 
    // FlowElement包含有的子类有:FlowNode/FlowNode/Activity/Gateway/DataObject
    // FlowNode包含有的子类有:Activity/Gateway
    if (parsedElement instanceof FlowElement) {
      FlowElement currentFlowElement = (FlowElement) parsedElement;
      currentFlowElement.setId(elementId);
      currentFlowElement.setName(elementName);
      
      if (currentFlowElement instanceof FlowNode) {  //流程节点处理
        FlowNode flowNode = (FlowNode) currentFlowElement;
        flowNode.setAsynchronous(async);
        flowNode.setNotExclusive(notExclusive);
        
        // 活动节点处理
        if (currentFlowElement instanceof Activity) {
          Activity activity = (Activity) currentFlowElement;
          activity.setForCompensation(isForCompensation);
          if (StringUtils.isNotEmpty(defaultFlow)) {
            activity.setDefaultFlow(defaultFlow);
          }
        }
        
        // 网关处理
        if (currentFlowElement instanceof Gateway) {
          Gateway gateway = (Gateway) currentFlowElement;
          if (StringUtils.isNotEmpty(defaultFlow)) {
            gateway.setDefaultFlow(defaultFlow);
          }
        }
      }
      
      // 数据节点处理
      if (currentFlowElement instanceof DataObject) {
        if (!activeSubProcessList.isEmpty()) {
          SubProcess subProcess = activeSubProcessList.get(activeSubProcessList.size() - 1);
          subProcess.getDataObjects().add((ValuedDataObject) parsedElement);
        } else {
          activeProcess.getDataObjects().add((ValuedDataObject) parsedElement);
        }
      }

      if (!activeSubProcessList.isEmpty()) {
        SubProcess subProcess = activeSubProcessList.get(activeSubProcessList.size() - 1);
        subProcess.addFlowElement(currentFlowElement);
      } else {
        activeProcess.addFlowElement(currentFlowElement);
      }
    }  // end if (parsedElement instanceof FlowElement) 
      
  } //end convertToBpmnModel
  
  
}
```
### (8).UserTaskXMLConverter
```
public class UserTaskXMLConverter extends BaseBpmnXMLConverter {
    protected Map<String, BaseChildElementParser> childParserMap = new HashMap<String, BaseChildElementParser>();
    
    protected static final List<ExtensionAttribute> defaultUserTaskAttributes = 
      Arrays.asList(
          new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_FORM_FORMKEY), 
          new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_DUEDATE), 
          new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_BUSINESS_CALENDAR_NAME),
          new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_ASSIGNEE), 
         new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_OWNER), 
         new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_PRIORITY), 
         new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CANDIDATEUSERS), 
         new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CANDIDATEGROUPS),
         new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CATEGORY),
         new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_SERVICE_EXTENSIONID),
         new ExtensionAttribute(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_SKIP_EXPRESSION)
  );
  
  public UserTaskXMLConverter() {
    HumanPerformerParser humanPerformerParser = new HumanPerformerParser();
    childParserMap.put(humanPerformerParser.getElementName(), humanPerformerParser);
    PotentialOwnerParser potentialOwnerParser = new PotentialOwnerParser();
    childParserMap.put(potentialOwnerParser.getElementName(), potentialOwnerParser);
    CustomIdentityLinkParser customIdentityLinkParser = new CustomIdentityLinkParser();
    childParserMap.put(customIdentityLinkParser.getElementName(), customIdentityLinkParser);
  }
  
  // 转换xml到Element
  protected BaseElement convertXMLToElement(XMLStreamReader xtr, BpmnModel model) throws Exception {
    // 解析:formKey
    String formKey = xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_FORM_FORMKEY);
    
    UserTask userTask = null;
    if (StringUtils.isNotEmpty(formKey)) {
      if (model.getUserTaskFormTypes() != null 
         && model.getUserTaskFormTypes().contains(formKey)) {
        //
        userTask = new AlfrescoUserTask();
      }
    }
    
    if (userTask == null) {
      // 创建UserTask
      userTask = new UserTask();
    }
    // 设置usertask的位置
    BpmnXMLUtil.addXMLLocation(userTask, xtr);
    // 解析:dueDate
    userTask.setDueDate(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_DUEDATE));
    // 解析:businessCalendarName
    userTask.setBusinessCalendarName(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_BUSINESS_CALENDAR_NAME));
    // 解析:category
    userTask.setCategory(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CATEGORY));
    // 设置forKey
    userTask.setFormKey(formKey);
    // 解析:assignee
    userTask.setAssignee(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_ASSIGNEE)); 
    // 解析:owner
    userTask.setOwner(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_OWNER));
    // 解析:priority
    userTask.setPriority(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_PRIORITY));
    
    // 解析:candidateUsers
    if (StringUtils.isNotEmpty(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CANDIDATEUSERS))) {
      String expression = xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CANDIDATEUSERS);
      userTask.getCandidateUsers().addAll(parseDelimitedList(expression));
    } 
    
    // 解析:candidateGroups
    if (StringUtils.isNotEmpty(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CANDIDATEGROUPS))) {
      String expression = xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_CANDIDATEGROUPS);
      userTask.getCandidateGroups().addAll(parseDelimitedList(expression));
    }
    
    // 解析:extensionId
    userTask.setExtensionId(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_SERVICE_EXTENSIONID));
    // 解析:skipExpression
    if (StringUtils.isNotEmpty(xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_SKIP_EXPRESSION))) {
      String expression = xtr.getAttributeValue(ACTIVITI_EXTENSIONS_NAMESPACE, ATTRIBUTE_TASK_USER_SKIP_EXPRESSION);
      userTask.setSkipExpression(expression);
    }

    // 解析自定义的信息(activiti:author="Hello World!!!")
    BpmnXMLUtil.addCustomAttributes(xtr, userTask, defaultElementAttributes, 
        defaultActivityAttributes, defaultUserTaskAttributes);

    // 解析子节点
    parseChildElements(getXMLElementName(), userTask, childParserMap, model, xtr);
    return userTask;
  } //end convertXMLToElement
    
}
```
### (9).解析<sequceFlow>
```
/**
* <sequenceFlow id="flow1" sourceRef="startevent1" targetRef="usertask1">
* </sequenceFlow>
*/

public class SequenceFlowXMLConverter extends BaseBpmnXMLConverter {
    protected BaseElement convertXMLToElement(XMLStreamReader xtr, BpmnModel model) throws Exception {
        SequenceFlow sequenceFlow = new SequenceFlow();
        BpmnXMLUtil.addXMLLocation(sequenceFlow, xtr);
        // sourceRef
        sequenceFlow.setSourceRef(xtr.getAttributeValue(null, ATTRIBUTE_FLOW_SOURCE_REF));
        // targetRef
        sequenceFlow.setTargetRef(xtr.getAttributeValue(null, ATTRIBUTE_FLOW_TARGET_REF));
        // name
        sequenceFlow.setName(xtr.getAttributeValue(null, ATTRIBUTE_NAME));
        // skipExpression
        sequenceFlow.setSkipExpression(xtr.getAttributeValue(null, ATTRIBUTE_FLOW_SKIP_EXPRESSION));
        // 解析子节点
        parseChildElements(getXMLElementName(), sequenceFlow, model, xtr);
        return sequenceFlow;
  } //end convertXMLToElement
  
}
```
### (10).通用子节点解析(BpmnXMLUtil)
>  这里所说的通用子节点解析是指:
>   <documentation> 
>   <extensionElements> 
>   <conditionExpression> 
>   <multiInstanceLoopCharacteristics>

```
public class BpmnXMLUtil implements BpmnXMLConstants {
    private static Map<String, BaseChildElementParser> genericChildParserMap = 
                                new HashMap<String, BaseChildElementParser>();
    // 定义通用解析
    static {
        addGenericParser(new ActivitiEventListenerParser());
        addGenericParser(new CancelEventDefinitionParser());
        addGenericParser(new CompensateEventDefinitionParser());
        addGenericParser(new ConditionExpressionParser());
        addGenericParser(new DataInputAssociationParser());
        addGenericParser(new DataOutputAssociationParser());
        addGenericParser(new DataStateParser());
        addGenericParser(new DocumentationParser());
        addGenericParser(new ErrorEventDefinitionParser());
        addGenericParser(new ExecutionListenerParser());
        addGenericParser(new FieldExtensionParser());
        addGenericParser(new FormPropertyParser());
        addGenericParser(new IOSpecificationParser());
        addGenericParser(new MessageEventDefinitionParser());
        addGenericParser(new MultiInstanceParser());
        addGenericParser(new SignalEventDefinitionParser());
        addGenericParser(new TaskListenerParser());
        addGenericParser(new TerminateEventDefinitionParser());
        addGenericParser(new TimerEventDefinitionParser());
        addGenericParser(new TimeDateParser());
        addGenericParser(new TimeCycleParser());
        addGenericParser(new TimeDurationParser());
        addGenericParser(new FlowNodeRefParser());
        addGenericParser(new ActivitiFailedjobRetryParser());
        addGenericParser(new ActivitiMapExceptionParser());
  } // end static                                

}
```

### (11).SequenceFlowXMLConverter  解析子节点
```
public class SequenceFlowXMLConverter extends BaseBpmnXMLConverter {
    protected BaseElement convertXMLToElement(XMLStreamReader xtr, BpmnModel model) throws Exception {
        // .... ...
        // *******************************************
        // 解析子节点
        parseChildElements(getXMLElementName(), sequenceFlow, model, xtr);
        // .... ...
        return sequenceFlow;
      } // end convertXMLToElement
}
```
### (12).BaseBpmnXMLConverter 解析子节点
```
public abstract class BaseBpmnXMLConverter implements BpmnXMLConstants {
    // 解析子节点
    protected void parseChildElements(String elementName, BaseElement parentElement, BpmnModel model, XMLStreamReader xtr) throws Exception {
        parseChildElements(elementName, parentElement, null, model, xtr);
    } //end parseChildElements
    
    // 解析子节点,并传入扩展的:BaseChildElementParser集合
    protected void parseChildElements(String elementName, BaseElement parentElement, Map<String, BaseChildElementParser> additionalParsers, BpmnModel model, XMLStreamReader xtr) throws Exception {
        Map<String, BaseChildElementParser> childParsers = new HashMap<String, BaseChildElementParser>();
        if (additionalParsers != null) {
          childParsers.putAll(additionalParsers);
        }
        // ******************************************8
        // 解析子节点
        BpmnXMLUtil.parseChildElements(elementName, parentElement, xtr, childParsers, model);
  } //end parseChildElements
}
```
### (13).BpmnXMLUtil.parseChildElements
```
public class BpmnXMLUtil implements BpmnXMLConstants {
    // ... ...
    
    // 解析子节点
    public static void parseChildElements(
        String elementName, 
        BaseElement parentElement, 
        XMLStreamReader xtr, 
        Map<String, BaseChildElementParser> childParsers, 
        BpmnModel model
     ) throws Exception {
         
    // genericChildParserMap 为系统默认的解析器
    Map<String, BaseChildElementParser> localParserMap =
        new HashMap<String, BaseChildElementParser>(genericChildParserMap);
    if (childParsers != null) {
      // 添加(覆盖) BaseChildElementParser
      localParserMap.putAll(childParsers);
    }
    boolean inExtensionElements = false;
    boolean readyWithChildElements = false;
    while (readyWithChildElements == false && xtr.hasNext()) {
      xtr.next();
      if (xtr.isStartElement()) {
        if (ELEMENT_EXTENSIONS.equals(xtr.getLocalName())) {
          inExtensionElements = true;
        } else if (localParserMap.containsKey(xtr.getLocalName())) {
          BaseChildElementParser childParser = localParserMap.get(xtr.getLocalName());
          if (inExtensionElements && !childParser.accepts(parentElement)) {
            //  解析:ExtensionElement
            ExtensionElement extensionElement = BpmnXMLUtil.parseExtensionElement(xtr);
            parentElement.addExtensionElement(extensionElement);
            continue;
          }
          
          localParserMap.get(xtr.getLocalName()).parseChildElement(xtr, parentElement, model);
        } else if (inExtensionElements) {
          ExtensionElement extensionElement = BpmnXMLUtil.parseExtensionElement(xtr);
          parentElement.addExtensionElement(extensionElement);
        }
      } else if (xtr.isEndElement()) {
        if (ELEMENT_EXTENSIONS.equals(xtr.getLocalName())) {
          inExtensionElements = false;
        } else if (elementName.equalsIgnoreCase(xtr.getLocalName())) {
          readyWithChildElements = true;
        }
      }
    }
  } //end parseChildElements
    
}
```
### (14).扩展节点解析案例
```
<process id="process1" name="My process" isExecutable="true" xmlns:activiti="http://activiti.org/bpmn" activiti:author="lixin">
  	<extensionElements>
        <activiti:field name="name" stringValue="lixin"/>
	</extensionElements>
</process> 
```
### (15).BpmnXMLConverter 
```
public class BpmnXMLConverter implements BpmnXMLConstants {
    // 扩展元素解析
    protected ExtensionElementsParser extensionElementsParser = new ExtensionElementsParser();
    
    public BpmnModel convertToBpmnModel(XMLStreamReader xtr) {
        // ...
        } else if (ELEMENT_EXTENSIONS.equals(xtr.getLocalName())) {
            // *****************************
            extensionElementsParser.parse(xtr, activeSubProcessList, activeProcess, model);
        } 
        // ...
    }
}
```
### (16).ExtensionElementsParser
```
public class ExtensionElementsParser implements BpmnXMLConstants {
    public void parse(XMLStreamReader xtr, List<SubProcess> activeSubProcessList, Process activeProcess, BpmnModel model) throws Exception {
    BaseElement parentElement = null;
    if (!activeSubProcessList.isEmpty()) {
      parentElement = activeSubProcessList.get(activeSubProcessList.size() - 1);

    } else {
      parentElement = activeProcess;
    }

    boolean readyWithChildElements = false;
    while (readyWithChildElements == false && xtr.hasNext()) {
      xtr.next();
      if (xtr.isStartElement()) {
        if (ELEMENT_EXECUTION_LISTENER.equals(xtr.getLocalName())) { // executionListener
          new ExecutionListenerParser().parseChildElement(xtr, parentElement, model);
        } else if (ELEMENT_EVENT_LISTENER.equals(xtr.getLocalName())) {  // eventListener
          new ActivitiEventListenerParser().parseChildElement(xtr, parentElement, model);
        } else if (ELEMENT_POTENTIAL_STARTER.equals(xtr.getLocalName())) { // potentialStarter
          new PotentialStarterParser().parse(xtr, activeProcess);
        } else {
          // extensionElements
          ExtensionElement extensionElement = BpmnXMLUtil.parseExtensionElement(xtr);
          parentElement.addExtensionElement(extensionElement);
        }

      } else if (xtr.isEndElement()) {
        if (ELEMENT_EXTENSIONS.equals(xtr.getLocalName())) {
          readyWithChildElements = true;
        }
      }
    }
  } // end parse
}
```
