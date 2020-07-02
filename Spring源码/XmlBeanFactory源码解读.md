# `XmlBeanFactory`源码解读

`XmlBeanFactory`是Spring专门用于读取`xml`源的`BeanFactory`。是`DefaultListableBeanFactory`的拓展类。这个类目前已经被标记为过期，然而对于学习Spring而言，恰到好处。

整个`XmlBeanFactory`只有两个构造函数，我们的话主要关注于`XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory)`。

```java
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
	super(parentBeanFactory);
       // 将加载BeanDefinition的工作交给reader去执行
       // 这里的Reader对应的是XmlBeanDefinitionReader类
       // XmlBeanDefinitionReader实现了BeanDefinitionReader接口
	this.reader.loadBeanDefinitions(resource);
}
```

跟进`XmlBeanDefinitionReader`的`loadBeanDefinitions`方法

```java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
   return loadBeanDefinitions(new EncodedResource(resource));
}
```

可以发现，`loadBeanDefinitions`调用了它的重载方法。

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   Assert.notNull(encodedResource, "EncodedResource must not be null");
   if (logger.isTraceEnabled()) {
      logger.trace("Loading XML bean definitions from " + encodedResource);
   }
   // 主要是用于判断传入的EncodedResource是否已经被加载了
   // 如果被加载了，则抛出异常
   Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

   if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
   }
	
   try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
      InputSource inputSource = new InputSource(inputStream);
      if (encodedResource.getEncoding() != null) {
         inputSource.setEncoding(encodedResource.getEncoding());
      }
      // 真正加载bean的方法 
      return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
   }
   finally { 
      currentResources.remove(encodedResource);
      if (currentResources.isEmpty()) {
         this.resourcesCurrentlyBeingLoaded.remove();
      }
   }
}
```

得到输入源后，执行真正负责加载Bean的方法`doLoadBeanDefinitions`

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
      throws BeanDefinitionStoreException {

   try {
       // 解析xml文件，得到DOM document对象
       // 这个方法是调用了javax.xml相关的解析方法
      Document doc = doLoadDocument(inputSource, resource);
      // 注册Bean定义，很关键的方法
      // count返回的是注册了多少个bean 
      int count = registerBeanDefinitions(doc, resource);
      if (logger.isDebugEnabled()) {
         logger.debug("Loaded " + count + " bean definitions from " + resource);
      }
      return count;
   }
   catch (BeanDefinitionStoreException ex) {
      throw ex;
   }
   catch (SAXParseException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
   }
   catch (SAXException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "XML document from " + resource + " is invalid", ex);
   }
   catch (ParserConfigurationException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Parser configuration exception parsing XML from " + resource, ex);
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "IOException parsing XML document from " + resource, ex);
   }
   catch (Throwable ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Unexpected exception parsing XML document from " + resource, ex);
   }
}
```

注册bean相关的方法`registerBeanDefinitions`

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // BeanDefinitionDocumentReader的作用是将XML中包含的Bean实体转换为BeanDefinition类
    // 默认实现是DefaultBeanDefinitionDocumentReader
   BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
   // 计算之前注册的Bean数量
   int countBefore = getRegistry().getBeanDefinitionCount();
   // 解析Document，并且注册到IOC容器中
   documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
   return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

一直到`registerBeanDefinitions`方法，属于`XmlBeanDefinitionReader`也结束了。下面我们需要探索的是，`DefaultBeanDefinitionDocumentReader`。这个类负责解析XML的节点信息，并且将这些信息转换成`BeanDefinition`信息。

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
   // 默认readConteng是XmlReaderContext 
   this.readerContext = readerContext;
   // 真正的注册方法 
   doRegisterBeanDefinitions(doc.getDocumentElement());
}
```

```java
protected void doRegisterBeanDefinitions(Element root) {
       // 任何嵌套<beans>的元素都会导致此方法被递归调用。
       // 针对于上一句的解释。parseBeanDefinitions 方法中会调用 parseDefaultElement 方法。
       // 而parseDefaultElement 中如果element的名称等于beans，则会回调此方法
       // 为了保证<beans>标签默认的数据的正确性，会将当前的delegate变量保存下来，这个变量可能为空。
       // 创建一个与之前有关联的新的delegate变量（此处的相关，查看源码后发现，只是将之前的默认值合并到新的delegate中），为了后续可以进行回退，
       // 最终回退this.delegate到它最初的引用
       // 这个行为排除掉没有必要的delegateNTest

   BeanDefinitionParserDelegate parent = this.delegate;
   		// 这里会设置一些默认值，包括autowire,lazy-init等等
       // 因为递归的原因，所以为了保证delegate的准确，引入了parent
       // 调用createDelegate是传入parent，如果parent不为空，那么创建的delegate会继承parent的默认属性
   this.delegate = createDelegate(getReaderContext(), root, parent);
       // 处理profile标签
   if (this.delegate.isDefaultNamespace(root)) {
      String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
      if (StringUtils.hasText(profileSpec)) {
         String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
               profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
         // We cannot use Profiles.of(...) since profile expressions are not supported
         // in XML config. See SPR-12458 for details.
         if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
            if (logger.isDebugEnabled()) {
               logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                     "] not matching: " + getReaderContext().getResource());
            }
            return;
         }
      }
   }
   // 空实现
   preProcessXml(root);
   // 解析并且注册的核心方法
   parseBeanDefinitions(root, this.delegate);
   // 空实现
   postProcessXml(root);

   this.delegate = parent;
}
```

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
   if (delegate.isDefaultNamespace(root)) {
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
         Node node = nl.item(i);
         if (node instanceof Element) {
            Element ele = (Element) node;
            if (delegate.isDefaultNamespace(ele)) {
                // 解析默认的元素，就是bean，import，alias，beans元素
               parseDefaultElement(ele, delegate);
            }
            else {
               delegate.parseCustomElement(ele);
            }
         }
      }
   }
   else {
      delegate.parseCustomElement(root);
   }
}
```

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
   if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
      importBeanDefinitionResource(ele);
   }
   else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
      processAliasRegistration(ele);
   }
   else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
       // 着重关注bean元素相关，这一块涉及ioc
      processBeanDefinition(ele, delegate);
   }
   else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
      // 注意，这个地方调用了doRegisterBeanDefinitions
      // 也就是doRegisterBeanDefinitions会发生递归的原因 
      doRegisterBeanDefinitions(ele);
   }
}
```

我们的关注点在bean。所以着重看`processBeanDefinition`方法

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 解析元素，封装为BeanDefinitionHolder，这个类里只有beanName,beanName,aliases
   BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
   if (bdHolder != null) {
       // 解析一些自定义属性
      bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
      try {
         // 注册到spring ioc中
         BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
      }
      catch (BeanDefinitionStoreException ex) {
         getReaderContext().error("Failed to register bean definition with name '" +
               bdHolder.getBeanName() + "'", ele, ex);
      }
      // Send registration event.
      getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
   }
}
```

这个方法堪称核心方法。包含了`xml` 到 `BeanDefinition`的过程，也包含了注册到`IOC`容器的过程。

我们优先看`parseBeanDefinitionElement`方法。

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
   return parseBeanDefinitionElement(ele, null);
}
```

喜闻乐见的重载调用。

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
   String id = ele.getAttribute(ID_ATTRIBUTE);
   String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
   // 获得别名集合
   List<String> aliases = new ArrayList<>();
   if (StringUtils.hasLength(nameAttr)) {
      String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
      aliases.addAll(Arrays.asList(nameArr));
   }

   String beanName = id;
   // 如果id为空，并且别名集合不为空，则将beanName赋值为aliases的第一个值
   if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
      beanName = aliases.remove(0);
      if (logger.isTraceEnabled()) {
         logger.trace("No XML 'id' specified - using '" + beanName +
               "' as bean name and " + aliases + " as aliases");
      }
   }
   // 保证beanName和aliases中的值都是唯一的，未被使用的
   if (containingBean == null) {
      checkNameUniqueness(beanName, aliases, ele);
   }
   // 根据传入的element进行解析，可能返回空
   AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
   if (beanDefinition != null) {
       // 如果beanName为空的时候，为Bean生成对应的名称
      if (!StringUtils.hasText(beanName)) {
         try {
            if (containingBean != null) {
               beanName = BeanDefinitionReaderUtils.generateBeanName(
                     beanDefinition, this.readerContext.getRegistry(), true);
            }
            else {
               beanName = this.readerContext.generateBeanName(beanDefinition);
               // Register an alias for the plain bean class name, if still possible,
               // if the generator returned the class name plus a suffix.
               // This is expected for Spring 1.2/2.0 backwards compatibility.
               String beanClassName = beanDefinition.getBeanClassName();
               if (beanClassName != null &&
                     beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                     !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                  aliases.add(beanClassName);
               }
            }
            if (logger.isTraceEnabled()) {
               logger.trace("Neither XML 'id' nor 'name' specified - " +
                     "using generated bean name [" + beanName + "]");
            }
         }
         catch (Exception ex) {
            error(ex.getMessage(), ele);
            return null;
         }
      }
      String[] aliasesArray = StringUtils.toStringArray(aliases);
      // 生成对应的实体类返回
      return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
   }

   return null;
}
```

不要看方法好像很长，总结一下其实也没有多少。

- 解析出bean元素的id和name属性，name属性标识bean的别名
- 创建一个`beanName`，将id的值赋给`beanName`
- 校验`beanName`和别名是否被使用，如果使用过了则跑出异常
- 对`xml`的节点进行解析，将配置的相关信息注入`BeanDefinition`中
- 判断`beanName`是否为空，如果为空的时候则创建一个属于这个Bean的名称
- 返回一个`BeanDefinitionHolder`对象，这个对象里包含`beanName`,别名，`BeanDefinition`

看吧，其实也不是很难。这里面唯一需要跟进的，就是`parseBeanDefinitionElement`方法。

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
      Element ele, String beanName, @Nullable BeanDefinition containingBean) {

   this.parseState.push(new BeanEntry(beanName));

   String className = null;
   if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
      className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
   }
   String parent = null;
   if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
      parent = ele.getAttribute(PARENT_ATTRIBUTE);
   }

   try {
       // 创建BeanDefinition，反射className对应的类
      AbstractBeanDefinition bd = createBeanDefinition(className, parent);
	  // 解析其他的属性	
      ...
      ...   
      return bd;
   }
   catch (ClassNotFoundException ex) {
      error("Bean class [" + className + "] not found", ele, ex);
   }
   catch (NoClassDefFoundError err) {
      error("Class that bean class [" + className + "] depends on not found", ele, err);
   }
   catch (Throwable ex) {
      error("Unexpected failure during bean definition parsing", ele, ex);
   }
   finally {
      this.parseState.pop();
   }

   return null;
}
```

我们回到`parseBeanDefinitionElement`。还记得吗？这个方法里有两个很关键的方法，第一个`delegate.parseBeanDefinitionElement(ele)`，第二个则是`BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry())`

准确的说，Spring提供给了一个`BeanDefinitionRegistry`接口专门用于注册bean信息。这个接口的默认实现是`DefaultListableBeanFactory`。

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
      throws BeanDefinitionStoreException {

   Assert.hasText(beanName, "Bean name must not be empty");
   Assert.notNull(beanDefinition, "BeanDefinition must not be null");

   if (beanDefinition instanceof AbstractBeanDefinition) {
      try {
         ((AbstractBeanDefinition) beanDefinition).validate();
      }
      catch (BeanDefinitionValidationException ex) {
         throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
               "Validation of bean definition failed", ex);
      }
   }

   BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
   if (existingDefinition != null) {
       // 不允许覆盖，则直接错
      if (!isAllowBeanDefinitionOverriding()) {
         throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
      }
      // 当传入的bean存在更大的权限的时候，则覆盖
      else if (existingDefinition.getRole() < beanDefinition.getRole()) {
         // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
         if (logger.isInfoEnabled()) {
            logger.info("Overriding user-defined bean definition for bean '" + beanName +
                  "' with a framework-generated bean definition: replacing [" +
                  existingDefinition + "] with [" + beanDefinition + "]");
         }
      }
      // 当传入的bean和缓存的bean不相等的时候
      else if (!beanDefinition.equals(existingDefinition)) {
         if (logger.isDebugEnabled()) {
            logger.debug("Overriding bean definition for bean '" + beanName +
                  "' with a different definition: replacing [" + existingDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      else {
         if (logger.isTraceEnabled()) {
            logger.trace("Overriding bean definition for bean '" + beanName +
                  "' with an equivalent definition: replacing [" + existingDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      this.beanDefinitionMap.put(beanName, beanDefinition);
   }
   else {
      if (hasBeanCreationStarted()) {
         // Cannot modify startup-time collection elements anymore (for stable iteration)
         synchronized (this.beanDefinitionMap) {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
            updatedDefinitions.addAll(this.beanDefinitionNames);
            updatedDefinitions.add(beanName);
            this.beanDefinitionNames = updatedDefinitions;
            removeManualSingletonName(beanName);
         }
      }
      else {
         // Still in startup registration phase
         this.beanDefinitionMap.put(beanName, beanDefinition);
         this.beanDefinitionNames.add(beanName);
         removeManualSingletonName(beanName);
      }
      this.frozenBeanDefinitionNames = null;
   }

   if (existingDefinition != null || containsSingleton(beanName)) {
      resetBeanDefinition(beanName);
   }
   else if (isConfigurationFrozen()) {
      clearByTypeCache();
   }
}
```

这个方法洋洋洒洒的写了几十行，这些我们都不太关心。我们真正关心的是`this.beanDefinitionMap.put(beanName, beanDefinition);`

而`this.beanDefinitionMap`是什么呢？

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```

其实只是一个简单的同步Map而已。

而上面代码的意思也很简单

- 对传入的`BeanDefinition`校验。这一步主要是为了标记重载方法
- 判断传入的`BeanDefinition`是否存在
  - 如果存在且覆盖标识为true的时候，则进行覆盖
  - 否则写入`beanDefinitionMap`和`beanDefinitionNames`
- 如果原先存在`BeanDefinition`，则重置给定bean的所有bean定义缓存，包括从中派生的bean缓存。

总结一下，整个`XmlBeanFactory`的流程如下：

- 读取XML源
- 解析DOCUMENT对象
- 将DOCUMENT对象解析为`BeanDefinition`
- 将`BeanDefinition`存入缓存中

而`SpringIOC`容器，本质上就是一个Map。

