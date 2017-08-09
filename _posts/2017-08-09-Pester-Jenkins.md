---
title: "Execute Pester tests from Jenkins"
date: 2017-08-09
---

# Execute Pester tests from Jenkins

## Background

Everytime I have to write a script I use the same approach. That approach has some similarities with __*Agile*__. Meaning; I write small blocks of code, test these blocks and _when everything works as aspected_ place the code into the script I'm developing.In Agile terms you can call this a __*sprint*__. I repeat these steps until my script is finished. Now, this works fine for me. Using this method I limit the possibilty of crappy code I don't need. And if a block of code doesn't work or needs to be adjusted, I can simply replace it with another one. 

### Pester / Test Driven Deployment

Pester is a completely different way to creating scripts. With Pester you first:

* Create a test that describes what it is you want to script.
* Run your test. (Which obviously failes).
* Create your script.
* Run the test again. (Which should lead to a success).
    
Now, I'm in the midst of learning Pester. So I'm not going in depth as there's still much to learn. Instead, I wil focus on Jenkins. 

### Jenkins

Jenkins is a true automation platform. Following the concepts of Continuous Integration / Testing and Deployment. As such, it is __**the**__ place to store and execute Pester tests. 

### Write the code

For this example I've created a very basic Pester test. A simple; _Describe_ that aspects the following output: _This is a very simple pester test!_.

```PowerShell
Describe "Get-PesterTest" {
    It "outputs 'This is a very simple pester test!'" {
        Get-PesterTest | Should Be 'This is a very simple pester test!'
    }
}
````

I saved this script as: _Get-PesterTest.Tests.ps1_ in: _C:\Temp\Pester\2_.

### Testing 1,2,3 - Is this thing on?

So, time to open up a PowerShell console. Change directory to: _C:\Temp\Pester\2_ and type:

```PowerShell
Invoke-Pester
```

This will invoke all pester tests in that particulair directory. There's only one in my case so it just executes the: _Get-PesterTest.Tests.ps1_. 

So, does it work!?

![Fail](https://codeinblue.files.wordpress.com/2017/08/p2.png)

Cleary not. As you see the test generated a failure. Obviously, since I didn't wrote the function. 

### Create the function

Now I know that my first test failed. (Which is good!) I can move on to actually build the function. That function should output: _This is a very simple pester test!_. 

```PowerShell
Function Get-PesterTest {
start-sleep -seconds 10
    "This is a very simple pester test!"
}
```

### Noticable stuff

* I've added a _start-Sleep -Seconds 10_ to make things a little more visible in Jenkins. 
* I've turned this function into a module: __Get-PesterTest__ and saved it to: _C:\Program Files\WindowsPowerShell\Modules\Get-PesterTest_. This makes the function available when I start a PowerShell console or when the Pester test is executed from the Jenkins server. 
 
Let's start a new PowerShell console and invoke the pester script.

```PowerShell
Invoke-Pester
```

![Success](https://codeinblue.files.wordpress.com/2017/08/p3.png)

That is looking promising. 

And my function works from the PowerShell Console.

![Works](https://codeinblue.files.wordpress.com/2017/08/p1.png)

### Jenkins time

This is where things are getting serious. Jenkins is a true automation platform and embraces the concepts of Continuous Testing/Integration and Deploying. So it makes sense storing and running Pester tests from the Jenkins platform. You can execute them from a single place and see the results on the Jenkins server webpage. And; because I recently added the Jenkins-Slack integration, I will receive all the information on my phone. 

#### But how to achieve this:

1. Login to Jenkins.

2. Click: _New Item_.

    ![New-Item](https://codeinblue.files.wordpress.com/2017/08/p4.png)

3. Enter the name for the new item. In my case: _Pester_Get-PesterTest_. Make sure you select: _Freestyle project_.

    ![Name](https://codeinblue.files.wordpress.com/2017/08/p5.png)

4. Browse to the _**Build**_ section, click: _Add build step_ and choose: _Windows PowerShell_.

    ![Build Actions](https://codeinblue.files.wordpress.com/2017/08/p6.png)

5. In the command window enter:

    ```PowerShell
    Invoke-Pester "PathToPesterFolder"
    ```

    ![Command](https://codeinblue.files.wordpress.com/2017/08/p7.png)

6. Click _Save_ in order the save the project and go back to the main screen.

7. On the main screen, click _Run_ to execute the task.

    ![ff](https://codeinblue.files.wordpress.com/2017/08/p8.png)

8. As soon as the task starts running it will appear on the left side of the screen. (Remember the _Start-Sleep_ Otherwise, it will build a little to fast to notice.)

    ![Running](https://codeinblue.files.wordpress.com/2017/08/p9.png)

9. When the task is complete, click the buildnumber and select: _Console Output_.

    ![Completetask](https://codeinblue.files.wordpress.com/2017/08/p10.png)

10. In the _Console Output_ window you will see the same output as on PowerShell Console.

    ![ConsoleOutput](https://codeinblue.files.wordpress.com/2017/08/p11.png)

### You've all done very well!!!

So far for a _Passed_ test. But what if a test fails? Well, let's see what happens!

To demonstrate this I've made some adjustments to my: _Get-PesterTest_ module function.

```PowerShell
function Get-PesterTest {
start-sleep -seconds 10
    "This will make a test fail!"
}
```

Remember the output of the test has to be: _This is a very simple pester test!_. So to make the test fail, simply change the output to: _This will make a test fail!_.

11. Go back to Jenkins and run the task again.

12. Open the _Console Output_ to see the results.

    ![Output2](https://codeinblue.files.wordpress.com/2017/08/p12.png)

    Now, the test indicates a fail.

### But we're not there yet!

Notice that the failed test also shows a _Success_. That's not good. A failed test should come up as a _Fail_. To accomplish that; we need to make a minor adjustment to the PowerShell command.

13. Open the task, click _Configure_ and browse to the: _Build_ section.

14. In the _Command_ textbox, add the folowing line: _-EnableExit_.

    ![Adjust](https://codeinblue.files.wordpress.com/2017/08/p13.png)

    The _EnabaleExit_ switch is used to _fail_ a build whenever a test has failed.

15. Save the task and run it again. This time when we open the _Console Output_ a real _failure_ will be displayed.

    ![RunAgain](https://codeinblue.files.wordpress.com/2017/08/p14.png)

## Wrapping up

This is just a very easy example of how to incorporate Pester on the Jenkins platform. But it gives a little insight into the possibilities of Jenkins and Pester.

[Go back](https://mufana.github.io/blog)