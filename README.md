# hibernate-view-safe-fix
The project represents and  demonstrates an interestig approach to fixing  LazyInitializationException in Spring Repository and JPA generally.
 - The approach is very simple and tries solving the problem in it sroot.  LazyInitializationException  is caused by attempt to hit database (execute query) on detached proxies after  a Hibernate session. The case is very usual in MVC and REST API planning.  The typical schenario is like this: an Entyity class has a mapping (let us say one to many), where collection of referred objects is joined by and external key in table of collected objects. By default Hibernate objects initialized  as lazy: Hibernate creates proxies, which really hist database when mapped fields are requiested. This way, any mapped collection consists of proxy objects, which have underlayed queries in  Hibernate session. When a session closed, JDBC queries are unlinked and the proxy object, which must be updated, cannot be updated. The LazyInitialization exception exactly signalizes about this case.
-  Direct use of Hibernate API does not lead to such collisions, because session is opened by means of calling related API function and is closed same way, when API caller will not close a Hibernate session till a connected object is still needed.But Spring's  JPA automation, which is used in Spring's data repository under the hood, works in conjunction with JTA and has the following hierarchy of beans dependency: transaction manager<- entity manager factory<datasources+hibernate udapter. Spring authomation is able to do the dependencies build under the hood, when persistence data are provided in the application's yml or .properties file. The same can be done manually, when there is a reason, both of the ways are exposed in examples.
- But anyway, either we use Spring's data repositories (by extending JpaRepository, when Spring builds underlaying worklfow), or when we use EntityManagher directly, Spring initializes and uses any EntityManagerFactoryBean's interface implementer, which produces EntityManager (JPA staff).
  Spring  JpaRepositories uses EntityManager on lower level of its interfaces methods execution and it closes it
   after related method of JpaRepository have been called. Closing the entity manager leads to closing Hibernate session.  The same mess will
    be happened in EJB when trying use the retuende data objects on View level.
   Anyway, after EntityManageris closes underlaying Hibernate session is closed also and all proxy objects are detached and any attempt to of operation, which requires a database hit will cause laxy initialation exception
    Reattachment is a worst of can be done because kills performance of databases.
 Another approach uses tricks of Hibernate JPA definition (see https://thoughts-on-java.org/LazyInitializationexception/?ck_subscriber_id=662673854).
  It is a good approach, but from point of view of system design and problem encapsulation it leads to delegation of the problem, which was birthed oustide Hibernate (Container behavior) to Hibernate. Is  a good working data model with properly working annotations must be extended because of external causes? Where are O-L of  the SOLID? 
 - We propose another approach: problem must be solved by those, who has created it, but using proper moment in environment. Delegate pattern togetherwith chain of responsibilties will work  fine. This way, we will subsclass the EntityManager and prevent call of its close() method where it is not desired. Instead of this, we will delegate the close operation to a component, which is called, for sure after all the possible usage of returned data objects are behind. The best place for this is a simple filter, and, preciselly more- after doFilter of filter chain is called. If we have an internal list to of delegate to run on this stage, then the task is solved safely, without involving Hibernate into  problems on upper layers.  
 - Fortunately Spring and Java provide the way to build the simple construction. EntityManager is produced by EntityManagerFactory, it is an interface and can be subclassed by Proxy even without CGLib. EntityManagerFactory is a bean and we intercept its initialization using related Spring API's event BeanPostPorcessor (see JPAHiberfixConfig#entityManagerProblemResolver(), JPAHiberfixConfig#generateEmProxyInterceptor(EntityManagerFactory bean) ). The decorator intercepts EntityManager creation and decorates EnityManager (see JPAHiberfixConfig#decorateEntityManager(EntityManager em)), where the decorator intercepts call of EntityManager#close and prevents  its immediate execution, when building a finalizing runnable (JPAHiberfix#addFinalizingRunnable). On this stage furthering usage  of entities is safe: Hibernate session is not closed and all calls of mapped entities will activate connected  query statements. The only topic of worrying is to close the EntityManager.
 - The interface com.jpa.config.common.ITransactionInterceptionManagement represents functionality of demarkate interception life time. It provides to appoint handling consumers which will he called when closing lifetime of interception. See com.jpa.config.common.ITransactionInterceptionManagement#finalizeInterceptedPostponedOperations(). The configuration class JPAHiberfixCongfig provides implementation of this interface, which calls runnables closing EntityManager (by calling the EnttyManager#close)
 - In the frames of Spring MVC/REST, we delegated call of EntityManager#close methods to implementer of the com.jpa.config.common.ITransactionInterceptionManagement in the flow  specially installed filter com.jpa.template.config.finalization.ConfigEntityManagementFinalization#loggingFilter. The Spring configuration  installs a filter, which just calls  com.jpa.config.common.ITransactionInterceptionManagement.finalizeInterceptedPostponedOperations() after calling filters chain. On this stage all possible uses cases of entities call are done.com.jpa.config.common.ITransactionInterceptionManagement provides sets if methods which enable start start and end interceptiing EntityManager calls, and even install handlers, which are called when EntityManager is being closed, and, additionally install handlers, which are called when transaction succeeded or  " vice versa" failed (compensator).
 - See JAPHiberfixConfig test  as example.
 


