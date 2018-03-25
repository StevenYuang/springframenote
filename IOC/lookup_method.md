Method injection
In most application scenarios, most beans in the container are singletons. When a singleton bean needs to collaborate with another singleton bean, or a non-singleton bean needs to collaborate with another non-singleton bean, you typically handle the dependency by defining one bean as a property of the other. A problem arises when the bean lifecycles are different. Suppose singleton bean A needs to use non-singleton (prototype) bean B, perhaps on each method invocation on A. The container only creates the singleton bean A once, and thus only gets one opportunity to set the properties. The container cannot provide bean A with a new instance of bean B every time one is needed.
A solution is to forego some inversion of control. You can make bean A aware of the container by implementing the ApplicationContextAware interface, and by making a getBean("B") call to the container ask for (a typically new) bean B instance every time bean A needs it. The following is an example of this approach:

// a class that uses a stateful Command-style class to perform some processing
package fiona.apple;
// Spring-API imports
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext; import org.springframework.context.ApplicationContextAware;
public class CommandManager implements ApplicationContextAware {
private ApplicationContext applicationContext;
public Object process(Map commandState) {
// grab a new instance of the appropriate Command
Command command = createCommand();
// set the state on the (hopefully brand new) Command instance command.setState(commandState);
return command.execute();
}
protected Command createCommand() {
// notice the Spring API dependency!
return this.applicationContext.getBean("command", Command.class);
}

public void setApplicationContext(
ApplicationContext applicationContext) throws BeansException {
this.applicationContext = applicationContext; }
}

The preceding is not desirable, because the business code is aware of and coupled to the Spring Framework. Method Injection, a somewhat advanced feature of the Spring IoC container, allows this use case to be handled in a clean fashion.

Lookup method injection
Lookup method injection is the ability of the container to override methods on container managed beans, to return the lookup result for another named bean in the container. The lookup typically involves a prototype bean as in the scenario described in the preceding section. The Spring Framework implements this method injection by using bytecode generation from the CGLIB library to generate dynamically a subclass that overrides the method.

Note
• For this dynamic subclassing to work, the class that the Spring bean container will subclass cannot be final, and the method to be overridden cannot be final either.
• Unit-testingaclassthathasanabstractmethodrequiresyoutosubclasstheclassyourself and to supply a stub implementation of the abstract method.
• Concretemethodsarealsonecessaryforcomponentscanningwhichrequiresconcreteclasses to pick up.
• Afurtherkeylimitationisthatlookupmethodswon’tworkwithfactorymethodsandinparticular not with @Bean methods in configuration classes, since the container is not in charge of creating the instance in that case and therefore cannot create a runtime-generated subclass on the fly.


Looking at the CommandManager class in the previous code snippet, you see that the Spring container will dynamically override the implementation of the createCommand() method. Your CommandManager class will not have any Spring dependencies, as can be seen in the reworked example:
package fiona.apple;
// no more Spring imports!
public abstract class CommandManager {
public Object process(Object commandState) {
// grab a new instance of the appropriate Command interface Command command = createCommand();
// set the state on the (hopefully brand new) Command instance command.setState(commandState);
return command.execute();
}
 // okay... but where is the implementation of this method?
protected abstract Command createCommand(); }


In the client class containing the method to be injected (the CommandManager in this case), the method to be injected requires a signature of the following form:

<public|protected> [abstract] <return-type> theMethodName(no-arguments);

If the method is abstract, the dynamically-generated subclass implements the method. Otherwise, the dynamically-generated subclass overrides the concrete method defined in the original class. For example:

<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype"> <!-- inject dependencies here as required -->
</bean>
<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager"> <lookup-method name="createCommand" bean="myCommand"/>
</bean>

The bean identified as commandManager calls its own method createCommand() whenever it needs a new instance of the myCommand bean. You must be careful to deploy the myCommand bean as a prototype, if that is actually what is needed. If it is as a singleton, the same instance of the myCommand bean is returned each time.

Alternatively, within the annotation-based component model, you may declare a lookup method through the @Lookup annotation:
    <public|protected> [abstract] <return-type> theMethodName(no-arguments);
 
<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype"> <!-- inject dependencies here as required -->
</bean>
<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager"> <lookup-method name="createCommand" bean="myCommand"/>
</bean>
 
public abstract class CommandManager {
public Object process(Object commandState) { Command command = createCommand(); command.setState(commandState);
return command.execute();
}
 @Lookup("myCommand")
protected abstract Command createCommand(); }


Or, more idiomatically, you may rely on the target bean getting resolved against the declared return type of the lookup method:
public abstract class CommandManager {
public Object process(Object commandState) { MyCommand command = createCommand(); command.setState(commandState);
return command.execute();
}
@Lookup
protected abstract MyCommand createCommand(); }
Note that you will typically declare such annotated lookup methods with a concrete stub implementation, in order for them to be compatible with Spring’s component scanning rules where abstract classes get ignored by default. This limitation does not apply in case of explicitly registered or explicitly imported bean classes.


