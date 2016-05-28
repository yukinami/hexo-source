title: 用Spring Test进行页面的功能测试
date: 2016-05-28 17:45:57
tags:
- SpringFramework
- 测试
---

## Spring Test

Spring Test框架提供的MockMvc可用于实现Controller的集成测试。

例如

```
MockHttpServletRequestBuilder createMessage = post("/messages/")
    .param("summary", "Spring Rocks")
    .param("text", "In case you didn't know, Spring Rocks!");

mockMvc.perform(createMessage)
    .andExpect(status().is3xxRedirection())
    .andExpect(redirectedUrl("/messages/123"));
```

<!-- more -->

如果想更新一步，对页面的功能功能测试，也是可以的

```
String summaryParamName = "summary";
String textParamName = "text";
mockMvc.perform(get("/messages/form"))
        .andExpect(xpath("//input[@name='" + summaryParamName + "']").exists())
        .andExpect(xpath("//textarea[@name='" + textParamName + "']").exists());

MockHttpServletRequestBuilder createMessage = post("/messages/")
        .param(summaryParamName, "Spring Rocks")
        .param(textParamName, "In case you didn't know, Spring Rocks!");

mockMvc.perform(createMessage)
        .andExpect(status().is3xxRedirection())
        .andExpect(redirectedUrl("/messages/123"));
```

但是这样的测试代码的有不少缺点，代码重复性较高，可维护性较差，页面稍微变更，现有的代码的xpath也不得不修改。 所以说并不适合用于页面功能测试


## 集成HtmlUnit

HtmlUnit是一个headless的浏览器，它可以由Webdriver来驱动。通过这种方式来进行页面功能测试，使测试更为充分，同时测试脚本可以给测试组复用，用[Selenium]进行完整的E2E测试。

Spring Test可以和HtmlUnit集成，进行页面功能的测试，同时对于Controller依赖的其他对象，又可以注入Mock对象。

```
HtmlPage createMsgFormPage = webClient.getPage("http://localhost/messages/form");

HtmlForm form = createMsgFormPage.getHtmlElementById("messageForm");
HtmlTextInput summaryInput = createMsgFormPage.getHtmlElementById("summary");
summaryInput.setValueAttribute("Spring Rocks");
HtmlTextArea textInput = createMsgFormPage.getHtmlElementById("text");
textInput.setText("In case you didn't know, Spring Rocks!");
HtmlSubmitInput submit = form.getOneHtmlElementByAttribute("input", "type", "submit");
HtmlPage newMessagePage = submit.click();

assertThat(newMessagePage.getUrl().toString()).endsWith("/messages/123");
String id = newMessagePage.getHtmlElementById("id").getTextContent();
assertThat(id).isEqualTo("123");
String summary = newMessagePage.getHtmlElementById("summary").getTextContent();
assertThat(summary).isEqualTo("Spring Rocks");
String text = newMessagePage.getHtmlElementById("text").getTextContent();
assertThat(text).isEqualTo("In case you didn't know, Spring Rocks!");
```


## 集成Webdriver

由于Webdriver是支持HtmlUnit的，因此，可以使用Webdriver代码驱动，好处是测试脚本可以给E2E测试复用。


下面是用了*Page Object Pattern*的测试页面

```
public class CreateMessagePage
        extends AbstractPage { 

    
    private WebElement summary;

    private WebElement text;

    
    @FindBy(css = "input[type=submit]")
    private WebElement submit;

    public CreateMessagePage(WebDriver driver) {
        super(driver);
    }

    public <T> T createMessage(Class<T> resultPage, String summary, String details) {
        this.summary.sendKeys(summary);
        this.text.sendKeys(details);
        this.submit.click();
        return PageFactory.initElements(driver, resultPage);
    }

    public static CreateMessagePage to(WebDriver driver) {
        // driver.get("http://localhost:8080/messages/form");
        get(driver, "messages/form");
        return PageFactory.initElements(driver, CreateMessagePage.class);
    }
}

```

测试逻辑

```
CreateMessagePage page = CreateMessagePage.to(driver);
ViewMessagePage viewMessagePage =
    page.createMessage(ViewMessagePage.class, expectedSummary, expectedText);
    
assertThat(viewMessagePage.getMessage()).isEqualTo(expectedMessage);
assertThat(viewMessagePage.getSuccess()).isEqualTo("Successfully created a new message");

```

### 关于*Page Object Pattern*

页面对象模式通过面向对象的方式把页面封装起来。无论是页面的元素，还是对页面的操作，都可以封装起来的，这大大提供了测试代码的复用率。 同时对页面的操作只需要通过调用方法即可实现，如同在浏览器页面上点选按钮一般简单，甚至对于一些复杂的流程操作，比直接在浏览器页面中操作更为高效。


## 集成Gab

Geb是一个Groovy的浏览器自动测试框架。它使用webdriver来驱动对页面的操作。可以把它理解为针对*Page Object Pattern*对Webdriver进行了更高层次的封装。利用它，可以更为方便的操作页面对象。同时它能够集成spock，能够很大程度的提高测试代码的质量。

下面是使用Geb进行页面功能测试的一个例子

```
class UserIndexPage extends Page {

    static url = "/users"
    static at = { title == "My website - Users" }
    static content = {
        menu { $("ul.nav.navbar-nav li", 2) }
        homeMenuButton { $("ul.nav.navbar-nav li:first-child a") }
        usersTable { $("table", 0) }
    }

}


class UserIndexIntegrationTests extends GebReportingSpec {

    @Autowired
    WebApplicationContext context

    def setup() {
        browser.driver = MockMvcHtmlUnitDriverBuilder
                .webAppContextSetup(context, springSecurity()).build()
    }

    def "go to users"() {
        when:
        to UserIndexPage

        then:
        at UserIndexPage

        expect:
        menu.hasClass("active")
        usersTable.find("tbody tr").size() == 3 //check record count

    }

    def "click home menu"() {
        when:
        to UserIndexPage

        then:
        homeMenuButton.click(HomePage)
    }
}
```


## 总结以及个人的一点看法

无论是集成HtmlUnit还是Geb，他们都是基于Spring Test的MockMVC的，也就是容器内的测试，它们都不会启动Servlet容器，因而测试效率是很高的。由于它并没有启动Servlet容器，所以页面必须是通过模板引擎来渲染的，如果页面的渲染是通过JSP forward来实现，那么测试时无法进行的， 也正因如此，他们并不是完整E2E测试方案，虽然他们的脚本确实可以提供给E2E测试。

Spring Test和HtmlUnit或者Geb的集成， 就可以当做Functional Test来做，不要对Controller的依赖对象进行Mock。如果需要，Controller的集成测试可以另外进行，然后注入mock对象。这样进行的功能测试对比完整的E2E测试有个很大的优势在于，每个页面的测试都会重新加载ApplicaitonContext初始化数据库环境，因而不同的页面测试的数据是互不影响的。不需要像E2E测试，额外地进行一些数据的准备工作。