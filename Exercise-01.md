# Declarative Syntax Basics

## Exercise 1.0 - Set-up
In this exercise you will setup a work environment for the lessons provided
in this workshop.  Ask the instructor for the URL of the server you will be using during the workshop.

### Create a Jenkins Account

1. Goto to the Workshop URL provided by the instructor;
2. Click on the **Create an account** link in the middle of the page under the **Login** button.
3. Complete the **Sign up** form (all fields are required) and click the **Sign up** button;
4. You should see a **Success** page - click on **the top page** link;

### Create a Team Master

Next, everyone will get their own Jenkins masters referred to as a Team Master and we will create, edit and interact with our Pipelines in Blue Ocean.

1. If not in Blue  Ocean, click on the **Blue Ocean** link in the left menu;
2. Click on the **Create team** button on the right side of the screen;
3. **Name this team** - enter a name for your team - perhaps your first initial with your last name and then click **Next**;
4. **Choose an icon for this team** - select an icon and color for your team and then click **Next**;
5. **Add people to this team** - your user will show up as a **Team Admin** and we won't be adding any additional users, but feel free to look around and then click **Next**;
6. **Select team master creation recipe** - click on the drop-down and select the **API Development** recipe;
7. Finally, click the **Create team** button.

## Exercise 1.1 - Basic Declarative Syntax Structure

In **Exercise 1.1** we will create a simple declarative pipeline directly within the Jenkins interface.

Declarative Pipelines must be enclosed within a `pipeline` block that in turn contains exactly one `stages` block. The `stages` block must have at least one `stage` block but can have an unlimited number of additional stages. Each `stage` block must have exactly one `steps` block.

Using the personal folder created for you in Exercise 1.0, do the following:

1. Click on the **create new jobs** link.  Alternatively you can create a job by selecting **New Item** from the left menu and then step 2 below.
2. Type **SimplePipeline** into the **Enter an item name** text box, click on **Pipeline**, and then click on the **OK** button.
3. Copy and paste the following code into the **Pipeline Script** text box near the bottom of the page:

```
pipeline {
   agent any
    
   stages {
      stage('Say Hello') {
         steps {
            echo 'Hello World!'   
            sh 'java -version'
         }
      }
   }
}
```

4. Click on **Save** and then click on **Build Now** in the left menu to run your pipeline.
5. Click on the blue dot for the job in **Build History** to view the console output from the job.  You should see Finished:  SUCCESS at the bottom of the output.

## Exercise 1.2 - Agent Labels

In **Exercise 1.2** we will update the pipeline we created in Exercise 1.1 to use a specific `agent` using the `label` syntax. As you saw from the build logs of the previous exercise, the Java version of the `agent any` was less than 9. We want to update our pipeline to use a version 9 JDK by replacing the `any` parameter with a `label` parameter:

1. Replace the `agent any` declaration with the following `agent` declaration:

```
  agent {
    label 'jdk9'
  }
```

2. Execute your job by clicking on **Build Now** and check the Console Log. You should see the following output after the `sh` step:

```
openjdk version "9.0.4"
OpenJDK Runtime Environment (build 9.0.4+12-Debian-4)
OpenJDK 64-Bit Server VM (build 9.0.4+12-Debian-4, mixed mode)
```

In addition to `any` and `label` you may also specify `none` and no global agent will be allocated for the entire Pipeline run and each `stage` section will need to contain its own `agent` section.

Before going on to the next exercise let's revert our pipeline to using:

```
   agent any
```

## Exercise 1.3 - Agents with Docker

In **Exercise 1.3** we will update the pipeline we created in Exercise 1.1 to execute steps in a Docker container. To update the pipeline:

1. In the `steps` block replace the `sh 'java -version'` step with the following step:

```
  sh 'go version'
```

2. Execute your job by clicking on **Build Now** and check the Console Log. The build will fail with `make: not found`

3. Click on configure and update the ```agent``` portion of the pipeline to read:

```
   agent {
      docker { 
        image 'golang:1.10.1-alpine'
        label 'docker-cloud' 
      }
   }
```

4. Execute your job by clicking on **Build Now** and check the Console Log to see how Jenkins pulls the appropriate docker image and runs your build inside the container created from that image. 

5. Remove the `label 'docker-cloud'` line from your pipeline job and build the job again by clicking on **Build Now**.  The job should still work... but why?? Pipeline Model Definition is why.  The Pipeline Model Definition allows an administrator to define the default label a job should use if one is not specified with docker.  In this case the Pipeline Model Definition is already set to `docker-cloud`.  This means that you will get an agent running dockerd everytime you use the docker {} block so using label parameter is redundant unless you need a specific docker agent.

Before going on to the next exercise let's revert our pipeline to using:

```
   agent any
```

And remove the `sh 'go version'` step

## Exercise 1.4 - Kubernetes Agents

In this exercise you explore the Kubernetes Plugin and will update a Jenkinsfile to use the `podTemplate` and `container` directives. In exercise 1.3 we saw how to use the Docker directive, allowing you to run steps inside an arbitray Docker image. Behind the scenes, there had to be a Jenkins agent that was able to execute against a Docker daemon to run containers. Here we will be using the [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin). The plugin creates a [Kubernetes Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) for each agent requested by a Jenkins job, with at least one Docker container running as the a JNLP agent, and stops the pod and all containers after the build is complete.

1. Copy and paste the following code into the **Pipeline Script** text box near the bottom of the page:

```
pipeline {
  agent {
    kubernetes {
      label 'kubernetes'
      containerTemplate {
        name 'go'
        image 'golang:1.10.1-alpine'
        ttyEnabled true
        command 'cat'
      }
    }
  }
  stages {
    stage('golang in k8s') {
        steps {
            container('go') {
                sh 'go version'
            }
        }
    }
  }
}
```
**Note:** Notice the use of the **container** directive.  This tells Jenkins which container in a Pod to use for the steps in the stage.  In this exercise a single container (golang) was explicitly defined in the pipeline, however a second container is implicitly created to handle the JNLP communiction between Jenkins and the Pod.

```
pipeline {
  agent {
    kubernetes {
      label 'kubernetes'
      containerTemplate {
        name 'go'
        image 'golang:1.10.1-alpine'
        ttyEnabled true
        command 'cat'
      }
    }
  }
  stages {
    stage('golang in k8s') {
        steps {
            container('gcc') {
                sh 'go version'
            }
            container('jnlp') {
                sh 'java -version'
            }
        }
     }
  }
}
```
**Note:** In the above example you were able to execute a java command in the implicitly defined **jnlp** container in the Pod.  The JNLP container is a part of every Pod created by the Jenkins Kubernetes plugin.

## Exercise 1.5 - Environment Directive

For **Exercise 1.4** we are going to update our **SimplePipeline** job to demonstrate how to use the `environment` directive to set and use environment variables. We will also see how this directive supports a special helper method `credentials()`. access pre-defined Credentials by their identifier in the Jenkins environment.

At the top of the pipeline insert the following code between the ```agent``` and ```stages``` blocks:  

```
   environment {
      MY_NAME = 'Mary'
   }
```

Then update the ```echo 'Hello World!'``` line to read ```echo "Hello ${MY_NAME}!"``` and run your build again to view the results.  Notice the change from '' to "".  Using double quotes will trigger extrapolation of environment variables.

We can also use environmental variables to import credentials. To demonstrate we will add the following line to our ```environment``` block:

```TEST_USER = credentials('test-user')```

We will also add the following ```echo``` steps within the ```steps``` of the Say Hello ```stage```:

```
            echo "${TEST_USER_USR}"
            echo "${TEST_USER_PSW}"
```

**Note**: After executing the build look at the console output and make note of the fact that the credential user name and password are masked when output via the echo command.

## Exercise 1.6 - Parameters

In **Exercise 1.5** we will alter our pipeline to accept external input in the form of a Parameter.

At the top of your pipeline insert the following block of code between the ```environment``` and ```stages``` blocks:

```
   parameters {
      string(name: 'Name', defaultValue: 'whoever you are', 
	     description: 'Who should I say hi to?')
   }
```
Then update the ```echo "Hello ${MY_NAME}!'``` line to read ```echo "Hello ${params.Name}!"``` and run your build again to view the results.

**Note**: Jenkins UI won't update properly when you save the pipeline to show the ```Build with parameters``` option so you need to run a build, view the results, and then return to the project to see the updated option.
