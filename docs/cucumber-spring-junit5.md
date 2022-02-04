---
layout: post
title:  "Cucumber 7 + Junit 5 + Spring Boot"
date:   2022-02-02 22:14:21 +0300
categories: java, cucumber-java, junit5, spring-boot
---

# Configuring a Java test framework using Cucumber 7, Junit 5, and Spring Boot 

First of all, I would like to thank [Palash](https://palashray.com/author/bonophulo/) that inspired me for this post and provided an 
MVP of Spring + Cucumber + Junit5 which works (see [here](https://palashray.com/example-of-creating-cucumber-based-bdd-tests-using-junit5-and-spring-dependency-injection/) the original post).

## Dependencies and POM configuration

Starting from Cucumber 7, the development team introduced a BOM which makes the dependencies management easier. 
So did the Junit development team.
Besides Cucumber and Junit dependencies, let's add the Spring Boot parent POM dependency along with some utilities like Lombok, some logging (e.g., Log4J) and AssertJ.

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.5.5</version>
    <relativePath/>
</parent>

<properties>
    <org.projectlombok.version>1.18.22</org.projectlombok.version>
    <org.apache.logging.log4j.version>2.17.1</org.apache.logging.log4j.version>

    <org.junit.jupiter.version>5.8.2</org.junit.jupiter.version>
    <io.cucumber.version>7.2.2</io.cucumber.version>

    <assertj-core.version>3.22.0</assertj-core.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-bom</artifactId>
            <version>${io.cucumber.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.junit</groupId>
            <artifactId>junit-bom</artifactId>
            <version>${org.junit.jupiter.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${org.projectlombok.version}</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>${org.apache.logging.log4j.version}</version>
    </dependency>

    <!-- Test dependencies -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-java</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-spring</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-junit-platform-engine</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.platform</groupId>
        <artifactId>junit-platform-suite</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>${assertj-core.version}</version>
    </dependency>
</dependencies>
```

There are quite many dependencies for a hello-world project, but what can we do...

The key parts in the given configuration represent the [io.cucumber:cucumber-junit-platform-engine](https://mvnrepository.com/artifact/io.cucumber/cucumber-junit-platform-engine) and [org.junit.platform:junit-platform-suite](https://mvnrepository.com/artifact/org.junit.platform/junit-platform-suite).
In Cucumber versions prior to 7 it was used `@Cucumber` annotation to mark the test runner.
Starting from Cucumber 7, this annotation is deprecated in favour of *JUnit Platform Suite*.

## Test Runner Configuration

The test runner is configured using *JUnit Platform Suite*

```java
import org.junit.platform.suite.api.ConfigurationParameter;
import org.junit.platform.suite.api.IncludeEngines;
import org.junit.platform.suite.api.SelectClasspathResource;
import org.junit.platform.suite.api.Suite;

import static io.cucumber.core.options.Constants.FILTER_TAGS_PROPERTY_NAME;
import static io.cucumber.core.options.Constants.GLUE_PROPERTY_NAME;
import static io.cucumber.core.options.Constants.PLUGIN_PUBLISH_QUIET_PROPERTY_NAME;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("bdd")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example.bdd")
@ConfigurationParameter(key = PLUGIN_PUBLISH_QUIET_PROPERTY_NAME, value = "true")
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME, value = "@RunMe and not @Skip")
public class RunCucumberTests {}
```

In case the configuration of your project is split across several packages and you need to add them to `GLUE`, specify the packages names as comma separated values.
The `@SelectClasspathResource("bdd")` annotation tells Cucumber to look for the feature files under *"classpath:bdd/"* folder or *"src/test/resources/bdd/"*.
Additional configuration parameters can be passed to Junit using `@ConfigurationParameter`.

## Spring integration

Our next step would be to tie the Spring context to our Cucumber test context. 
We can achieve this by combining the `@CucumberContextConfiguration` with our Spring test configuration.

```java
import com.example.config.SpringTestConfig;
import io.cucumber.spring.CucumberContextConfiguration;
import org.springframework.boot.test.context.SpringBootTest;

@CucumberContextConfiguration
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.NONE, 
    classes = SpringTestConfig.class
)
public class CucumberSpringContextConfig {}
```

We could implement our step definitions even here, although the best practices advise us to separate them to another file.
The `SpringTestConfig.class`, as one can deduce by the name, is the place where our Spring test configuration resides.

```java
import com.example.bdd.custom.CustomLogger;
import lombok.extern.log4j.Log4j2;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@Log4j2
@TestConfiguration
public class SpringTestConfig {

    @Bean
    public CustomLogger customLogger() {
        return new CustomLogger();
    }

    @Bean
    public RestTemplate restTemplate() {
        log.info("Creating new RestTemplate bean");
        return new RestTemplate();
    }
}
```

For example, in the given configuration we configure two beans which we can then inject into our step definition class.

## Step definitions

The step definition classes are seen by Spring as component classes.
Thus, we can inject other beans/components and use them.

The best practices advise us to split our step definitions according to their application domain.

```java
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;

@Log4j2
@RequiredArgsConstructor
public class AccessibilityStepDefinitions {

    @Given("The application is available at {string}")
    public void applicationBaseUrl(final String baseUrl) {
        log.info("Application is available at {}", baseUrl);
    }

    @Given("^Health check is fine at (.*)$")
    public void checkApplicationHealth(final String uri) {
        log.info("Application is healthy! {}", uri);
    }

    @Given("A valid connection to database is prepared")
    public void givenAValidConnectionToDatabaseIsPrepared() {
        log.info("Connection valid");
    }

    @When("The available connection to database is closing")
    public void whenTheAvailableConnectionToDatabaseIsClosing() {
        log.info("Connection closing");
    }

    @Then("The connection to the database is closed")
    public void thenTheConnectionToTheDatabaseIsClosed() {
        log.info("Connection closed");
    }
}
```

Additionally, we may have, for example, `DatabaseStepDefinitions` which would be used by Cucumber.
Having the above step definition, we may create the `src/test/resources/bdd/accessibility.feature` file with the following content:

```gherkin
Feature: Test our service's rest endpoints

  Background:
    Given The application is available at "http://localhost:8080"
    And Health check is fine at "/actuator/health"

  @RunMe
  Scenario: Open and close a connection to database 1-1
    Given A valid connection to database is prepared
    When The available connection to database is closing
    Then The connection to the database is closed
```

## References

1. Example of creating Cucumber based BDD tests using JUnit5 and Spring Dependency Injection, [https://palashray.com/example-of-creating-cucumber-based-bdd-tests-using-junit5-and-spring-dependency-injection/](https://palashray.com/example-of-creating-cucumber-based-bdd-tests-using-junit5-and-spring-dependency-injection/)
2. Cucumber Spring, [https://github.com/cucumber/cucumber-jvm/tree/main/spring](https://github.com/cucumber/cucumber-jvm/tree/main/spring)
3. Cucumber JUnit Platform Engine, [https://github.com/cucumber/cucumber-jvm/tree/main/junit-platform-engine](https://github.com/cucumber/cucumber-jvm/tree/main/junit-platform-engine)
