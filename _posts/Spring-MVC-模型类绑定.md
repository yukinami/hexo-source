title: Spring MVC 模型类绑定
date: 2016-12-06 13:37:42
tags:
- Spring MVC
- Spring Data
---

在使用Spring MVC的时候有一个很普通的需求，根据url中传递的id参数，查询并绑定绑定到一个实体对象

<!--more-->

```
@Controller
@RequestMapping("/users")
public class UserController {

  private final UserRepository userRepository;

  @Autowired
  public UserController(UserRepository userRepository) {
    Assert.notNull(repository, "Repository must not be null!");
    this.userRepository = userRepository;
  }

  @RequestMapping("/{id}")
  public String showUserForm(@PathVariable("id") Long id, Model model) {

    // Do null check for id
    User user = userRepository.findOne(id);
    // Do null check for user

    model.addAttribute("user", user);
    return "user";
  }
}
```

*Spring Data*提供的`DomainClassPropertyEditorRegistrar`能够自动地查找所有注册的*Spring Data repositories*，并使用它进行查询转换

```
<bean class="….web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">
  <property name="webBindingInitializer">
    <bean class="….web.bind.support.ConfigurableWebBindingInitializer">
      <property name="propertyEditorRegistrars">
        <bean class="org.springframework.data.repository.support.DomainClassPropertyEditorRegistrar" />
      </property>
    </bean>
  </property>
</bean>
```

上面的代码即可简化为

```
@Controller
@RequestMapping("/users")
public class UserController {

  @RequestMapping("/{id}")
  public String showUserForm(@PathVariable("id") User user, Model model) {

    model.addAttribute("user", user);
    return "userForm";
  }
}
```

同时使用了PropertyEditors转换后，还能够继续对User对象填充页面的提交的值，这在JPA更新的场合极为方便，省去了手动Merge的过程。
