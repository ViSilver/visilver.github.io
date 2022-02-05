---
layout: post
title:  "Cucumber 7 + JUnit 5 + Spring Boot with Multiple Test Configurations"
date:   2022-02-02 22:14:21 +0300
categories: java, cucumber-java, junit5, spring-boot
---

# Configuring Multiple Java Test Frameworks Using Cucumber 7, Junit 5, and Spring Boot 
{: .no_toc}

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .no_toc .text-delta }
1. TOC
{:toc}
</details>

In my previous [**post**]({{ '/cucumber-spring-junit5.html' | relative_url }}) I have described how to set up a Java test framework using Cucumber 7, Junit 5, and Spring Boot.
In this post, we will extend the previous configuration in order to set up multiple Cucumber test executions within the same project.
In order to do this, we will use Spring profiles.

## Spring Configurations Based on Profiles

The first step is to split the application configuration based on the desired profile. 
In order to illustrate the example, let's define two profiles: `profile1` and `profile2`.
Then our configurations can be:

```java
import lombok.extern.log4j.Log4j2;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Profile;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@Log4j2
@TestConfiguration
@Profile("profile1")
public class Profile1Config {

    @Bean
    public ExecutorService executorService() {
        log.info("Creating profile1 executor service");
        return Executors.newFixedThreadPool(2);
    }
}
```
```java
import lombok.extern.log4j.Log4j2;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Profile;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@Log4j2
@TestConfiguration
@Profile("profile2")
public class Profile2Config {

    @Bean
    public ExecutorService executorService() {
        log.info("Creating profile2 executor service");
        return Executors.newFixedThreadPool(2);
    }
}
```

Additionally, we may have a common Spring configuration classes. 
We may use the `SpringTestConfig.java` from our [previous setup]({{ '/cucumber-spring-junit5.html#spring-application-context-configuration' | relative_url }}).

## Cucumber Test Suites Configuration

The suite configuration consists of two parts: Cucumber Spring context configuration and the Cucumber test runner configuration.

### Cucumber Spring Context Configuration

The next step is to define the Cucumber Spring context configuration. 
We will create a separate package for each profile: `com.example.cucumberconfig.profile1` and `com.example.cucumberconfig.profile2`.
In each of these packages there will reside a Cucumber Spring config and a test runner.

The profile specific configs can be:
```java
package com.example.cucumberconfig.profile1;

import com.example.config.Profile1Config;
import com.example.config.SpringTestConfig;
import io.cucumber.spring.CucumberContextConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@CucumberContextConfiguration
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.NONE,
        classes = {SpringTestConfig.class, Profile1Config.class}
)
@ActiveProfiles("profile1")
public class CucumberSpringContextConfig {}
```
```java
package com.example.cucumberconfig.profile2;

import com.example.config.Profile2Config;
import com.example.config.SpringTestConfig;
import io.cucumber.spring.CucumberContextConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@CucumberContextConfiguration
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.NONE,
        classes = {SpringTestConfig.class, Profile2Config.class}
)
@ActiveProfiles("profile2")
public class CucumberSpringContextConfig {}
```

For each of the configuration we activate a specific profile (using `@ActiveProfiles` annotation) which filter out the necessary bean spawning and their injection.

### Cucumber Test Runner Configuration
In order to define our Cucumber test runners, we need to figure out how to tell Cucumber to run a subset of our scenarios and to ignore the others. 
We could, of course, use [Cucumber tags](https://cucumber.io/docs/cucumber/api/?sbsearch=junit%205#tags), but it is annoying to mark each our scenario with a tag.
One solution would be to split our feature files into profile-specific folders and to link them to the suite using `@SelectClasspathResource`.

Another problem is defining profile specific steps.
The solution to this would be to split the step definitions across profile specific classes and add them to `GLUE` path using `@ConfigurationParameter`.
Additionally to step definitions, we need to `glue` the Cucumber Spring context configuration class.
Fortunately, we can specify multiple packages in the `GLUE` path as comma-separated values.

Unfortunately, currently, Cucumber (7.2.2) does not have a `glueClass` feature of supporting dynamically adding the packages to `GLUE` path using the class name (there is a [bounty](https://github.com/cucumber/cucumber-jvm/issues/1707) for this feature, though).
We have to enumerate the packages manually.
This is error prone and inconvenient, especially when the name of the glued packages change.

The final test runner configurations are:
```java
package com.example.cucumberconfig.profile1;

import org.junit.platform.suite.api.ConfigurationParameter;
import org.junit.platform.suite.api.IncludeEngines;
import org.junit.platform.suite.api.SelectClasspathResource;
import org.junit.platform.suite.api.Suite;

import static io.cucumber.core.options.Constants.FILTER_TAGS_PROPERTY_NAME;
import static io.cucumber.core.options.Constants.GLUE_PROPERTY_NAME;
import static io.cucumber.core.options.Constants.PLUGIN_PUBLISH_QUIET_PROPERTY_NAME;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("bdd/profile1")
@ConfigurationParameter(
        key = GLUE_PROPERTY_NAME,
        value = "com.example.bdd.custom," +
                "com.example.bdd.stepdefs.common," +
                "com.example.bdd.stepdefs.profile1," +
                "com.example.cucumberconfig.profile1"
)
@ConfigurationParameter(key = PLUGIN_PUBLISH_QUIET_PROPERTY_NAME, value = "true")
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME, value = "@RunMe")
public class RunProfile1CucumberTests {}
```
```java
package com.example.cucumberconfig.profile2;

import org.junit.platform.suite.api.ConfigurationParameter;
import org.junit.platform.suite.api.IncludeEngines;
import org.junit.platform.suite.api.SelectClasspathResource;
import org.junit.platform.suite.api.Suite;

import static io.cucumber.core.options.Constants.FILTER_TAGS_PROPERTY_NAME;
import static io.cucumber.core.options.Constants.GLUE_PROPERTY_NAME;
import static io.cucumber.core.options.Constants.PLUGIN_PUBLISH_QUIET_PROPERTY_NAME;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("bdd/profile2")
@ConfigurationParameter(
        key = GLUE_PROPERTY_NAME,
        value = "com.example.bdd.custom," +
                "com.example.bdd.stepdefs.common," +
                "com.example.bdd.stepdefs.profile2," +
                "com.example.cucumberconfig.profile2"
)
@ConfigurationParameter(key = PLUGIN_PUBLISH_QUIET_PROPERTY_NAME, value = "true")
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME, value = "@RunMe")
public class RunProfile2CucumberTests {}
```

## Step Definitions

Due to the magical Cucumber-Spring integration, if the step definition classes reside under `GLUE` path, then they are added to context and transformed into Spring beans.
Thus, be careful what you `glue` and try to minimize your test application context.
Moreover, by gluing all step definitions and having profile specific bean injection into some particular step definition classes may corrupt the application context during startup because of the missing beans.
Always check the packages added to `GLUE` path in the test runner if any errors raise.

In our case, we will split our step definitions across packages so that we would be able to `glue` them.
Additionally, we have some step definitions common to both profiles.
Gluing them too.

Finally, our step definitions may look like this:
```java
package com.example.bdd.stepdefs.common;

import com.example.bdd.custom.CustomLogger;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;

@Log4j2
@RequiredArgsConstructor
public class DatabaseStepDefinitions {

    private final CustomLogger customLogger;

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
```java
package com.example.bdd.stepdefs.profile1;

import com.example.bdd.custom.CustomLogger;
import io.cucumber.java.en.Then;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.context.annotation.Profile;

import java.util.concurrent.ExecutorService;

@Log4j2
@RequiredArgsConstructor
@Profile("profile1")
public class Profile1StepDefinitions {

    private final ExecutorService executorService;

    private final CustomLogger customLogger;

    @Then("^A custom profile1 step is executed$")
    public void aCustomProfileStepIsExecuted() {
        log.info("Custom profile1 step execution");
    }
}
```
```java
package com.example.bdd.stepdefs.profile2;

import io.cucumber.java.en.Then;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.context.annotation.Profile;

import java.util.concurrent.ExecutorService;

@Log4j2
@RequiredArgsConstructor
@Profile("profile2")
public class Profile2StepDefinitions {

    private final ExecutorService executorService;

    @Then("^A custom profile2 step is executed$")
    public void aCustomProfileStepIsExecuted() {
        log.info("Custom profile2 step execution");
    }
}
```

The injection of the `executorService` and `customLogger` beans in this case is done to illustrate the profile-specific dependency injection by Spring.
Additionally, we have added a custom step for each of the profiles.


## Feature File and Scenarios

The scenarios are split across feature files and subfolders which are loaded by the test runner.
The common test scenarios can be located in a common subfolder which can be loaded as a separate resource to the test runner just like the profile specific feature files.

Finally, our feature files are as follows:
```gherkin
Feature: Test our service's rest endpoints for 'profile1'

  Background: The application is reachable and healthy
    Given The application is available at "http://localhost:8080"
    And Health check is fine at "/actuator/health"

  @RunMe
  Scenario: Open and close a connection to database 1-1
    Given A valid connection to database is prepared
    When The available connection to database is closing
    Then The connection to the database is closed
    And A custom profile1 step is executed
```

```gherkin
Feature: Test our service's rest endpoints for 'profile2'

  Background: The application is reachable and healthy
    Given The application is available at "http://localhost:8080"
    And Health check is fine at "/actuator/health"

  @RunMe
  Scenario: Open and close a connection to database 1-1
    Given A valid connection to database is prepared
    When The available connection to database is closing
    Then The connection to the database is closed
    And A custom profile2 step is executed
```

## File Structure

The entire project file structure is as follows:
```
.
├── pom.xml
└── src
    └── test
        ├── java
        │   └── com
        │       └── example
        │           ├── bdd
        │           │   ├── custom
        │           │   │   └── CustomLogger.java
        │           │   └── stepdefs
        │           │       ├── common
        │           │       │   └── DatabaseStepDefinitions.java
        │           │       ├── profile1
        │           │       │   └── Profile1StepDefinitions.java
        │           │       └── profile2
        │           │           └── Profile2StepDefinitions.java
        │           ├── config
        │           │   ├── Profile1Config.java
        │           │   ├── Profile2Config.java
        │           │   └── SpringTestConfig.java
        │           └── cucumberconfig
        │               ├── profile1
        │               │   ├── CucumberSpringContextConfig.java
        │               │   └── RunProfile1CucumberTests.java
        │               └── profile2
        │                   ├── CucumberSpringContextConfig.java
        │                   └── RunProfile2CucumberTests.java
        └── resources
            ├── bdd
            │   ├── profile1
            │   │   └── accessibility.feature
            │   └── profile2
            │       └── accessibility.feature
            ├── cucumber.properties
            └── log4j2.xml
```

## (Bonus) Maven Surfire Plugin Options

In order to configure the Cucumber pretty output when running the tests using Maven, here there are some options that might be handy:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.0.0-M5</version>
    <configuration>
        <properties>
            <configurationParameters>
                cucumber.junit-platform.naming-strategy=long
                cucumber.plugin=pretty,html:target/site/cucumber-pretty.html
                cucumber.publish.quiet=true
                cucumber.publish.enable=false
            </configurationParameters>
        </properties>
    </configuration>
</plugin>
```

More about Cucumber properties you can read [here](https://github.com/cucumber/cucumber-jvm/tree/main/junit-platform-engine#configuration-options).

More about Maven Surfire plugin configuration you can read [here](https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit-platform.html).

## References
1. Cucumber Spring, [https://github.com/cucumber/cucumber-jvm/tree/main/spring](https://github.com/cucumber/cucumber-jvm/tree/main/spring)
2. Cucumber JUnit Platform Engine, [https://github.com/cucumber/cucumber-jvm/tree/main/junit-platform-engine](https://github.com/cucumber/cucumber-jvm/tree/main/junit-platform-engine)
3. Spring Profiles, [https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)
4. Cucumber Tags, [https://cucumber.io/docs/cucumber/api/?sbsearch=junit%205#tags](https://cucumber.io/docs/cucumber/api/?sbsearch=junit%205#tags)
5. Cucumber Configuration Options, [https://github.com/cucumber/cucumber-jvm/tree/main/junit-platform-engine#configuration-options](https://github.com/cucumber/cucumber-jvm/tree/main/junit-platform-engine#configuration-options)
6. Maven Surfire JUnit 5, Configuration Parameters, [https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit-platform.html](https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit-platform.html)
