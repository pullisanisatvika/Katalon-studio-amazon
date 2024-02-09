Using AWS Java SDK for Amazon S3 in Katalon Studio --- how to resolve external dependencies with Gradle
===========

## What is this?

This is a small [Katalon Studio](https://www.katalon.com/) project for demonstration purpose.
You can download the zip file from the [Releases](https://github.com/kazurayam/UsingAwsSdkInKatalonStudio/releases) page, unzip it, and run in Katalon Studio on your PC.

This project is developed with Katalon Studio version 5.9.1.

This project is made to show you
1. how to resolve the external dependencies (a lot of jar files in the `<projectDir>/Drivers` directory) in a automated way using [Gradle](https://docs.gradle.org/current/userguide/introduction_dependency_management.html).
2. how to call *AWS Java SDK for Amazon S3* in a test case in Katalon Studio.

## Problems to solve

### (1) I want to use Amazon S3 in my test cases

My [*Visual Testing in Katalon Studio*](https://forum.katalon.com/t/visual-testing-in-katalon-studio/13361) project enabled me to do visual regression testing. It compares 2 URLs of my AUT (production & development) in a time-slicing manner.

Now I have a plan to develop another way of visual regression testing. New katalon project will do it chronologically. It will compare the current URL with the set of screen shots taken previously --- taken 3 hours ago, taken yesterday evening, or taken last week. This feature would enable me to check the system's stability before/after application upgrades. I would be able to automatically compare hundreds of pages against the previous images taken before the work.

But this new idea brings a blocking issue to me. I need to store a lot of versions of screen shots. Driving the chronological test in a Continuous Integration system will result huge number of screen shot files (over 10000 easily). How can I *manage* the accumulated screen shot files? One idea has come up to my mind. Why not using Cloud Storage service such as Amazon S3?

Cloud Storage seems to be promising for me. With it, I do not have to worry about the capacity, it's cheap, files older than 1 month will be automatically deleted, ... etc.

Therefore I want my Katalon Studio project calls *AWS Java SDK for S3* so that it can transport screenshot files to and from Amazon S3.

### (2) How to resolve external dependencies?

[AWS document](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/examples-s3.html) provides enough information how to use *AWS Java SDK for S3* in my Java/Groovy codes. So I tried to write a test case in Groovy in Katalon Studio which calls AWS API for S3. I wanted it to list my S3Buckets in my AWS account. Soon I encountered a blocking problem.

Katalon Studio requires all of the external dependencies (jar files) put into the `<projectDir>/Drivers` directory. I knew I need the `aws-java-sdk-s3-1.11.470.jar`. But there are a lot more dependencies (in fact 14 jars). Unfortunately Katalon Studio does not provide any dependency management.

A quote from [Gradle document](https://docs.gradle.org/current/userguide/introduction_dependency_management.html)
>What is dependency management?
>
>Software projects rarely work in isolation. In most cases, a project relies on reusable functionality in the form of libraries or is broken up into individual components to compose a modularized system. Dependency management is a technique for declaring, resolving and using dependencies required by the project in an automated fashion.

It looked impossible for me to look up all the necessary jar files for running a test case with AWS SDK in Katalon Studio.

## Solution

Use `com.katalon.gradle-plugin`. devalex88 informed of it at the following comment in the Katalon Forum:
- https://forum.katalon.com/t/katalon-studios-built-in-keywords-is-now-open-source-for-community-contribution/14281/12

`com.katalon.gradle-plugin` is a Gradle plugin. By executing
```
$ gradle katalonCopyDependencies
```
you can let Gradle download the specified jar file and other dependencies from the MaveCentral repository, and put them into the `<projectDir>/Drivers` directory. For example, I got:

```
$ ls Drivers
katalon_generated_aws-java-sdk-core-1.11.470.jar
katalon_generated_aws-java-sdk-kms-1.11.470.jar
katalon_generated_aws-java-sdk-s3-1.11.470.jar
katalon_generated_commons-codec-1.10.jar
katalon_generated_commons-logging-1.2.jar
katalon_generated_httpclient-4.5.5.jar
katalon_generated_httpcore-4.4.9.jar
katalon_generated_ion-java-1.0.2.jar
katalon_generated_jackson-annotations-2.6.0.jar
katalon_generated_jackson-core-2.6.7.jar
katalon_generated_jackson-databind-2.6.7.2.jar
katalon_generated_jackson-dataformat-cbor-2.6.7.jar
katalon_generated_jmespath-java-1.11.470.jar
katalon_generated_joda-time-2.8.1.jar
```

## Description

### Prerequisite

Here I assume you have some experience of AWS. You should already have an AWS account for you. And you already have prepared AWS credential info in the `<HOME>./aws` directory of your PC. I assume you have installed AWS CLI and you can successfuly do the following operation in the commandline:
```
$ aws s3 ls
2017-12-27 06:30:04 myBucket1
2016-03-31 23:11:02 myBucket2
2015-03-01 13:01:59 myBucket3
...
```
I am going to do the same operation (list my S3 Buckets) by a Katalon Test Case.


### Resolving dependencies

Here I assume you have got started with [Gradle](https://gradle.org/guides/). Installation guide is [here](https://gradle.org/install/).

I added [`<projectDir>/build.gradle`](build.gradle) file. In there I declared only one jar as dependency of the project.
```
dependencies {
    // https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk-s3
    compile group: 'com.amazonaws', name: 'aws-java-sdk-s3', version: '1.11.470'
}
```

You can find the jar in the Maven Central repository https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk-s3

I the command line, I executed the following command:
```
$ cd <katalon project path>
$ gradle katalonCopyDependencies
```

Gradle took a few minutes to finish the `katalonCopyDependencies` task. It downloaded 14 jar files into the local `<HOME>/.m2` diretory --- this took a few minutes. And the task copied necessary jar files into `<projectDir>/Drivers`.

I closed the project once, and reopened it in Katalon Studio. This was necessary in order to let Katalon Studio acknowledge the new jar files in the Drivers directory.

I wrote a test case [ListMyS3Buckets](Scripts/ListMyS3Buckets/Script1545180936915.groovy).

I ran the test case, and got the following output:

```
* My Amazon S3 buckets are:
* myBucket1
* myBucket2
* myBucket3
...
```

It worked.

### Note on .gitignore

If you are using git for the Katalon project, you should add the following line in the `.gitignore` file.
```
/Drivers
```
You should not include the external jar files downloaded from Maven repositories.
