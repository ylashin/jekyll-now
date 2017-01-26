---
title:  "SharePoint meets TensorFlow"
date:   2017-01-26 10:00:00
categories: [TensorFlow,SharePoint,.net core]
tags: [TensorFlow,SharePoint,.net core]
---

### Before they met
I guess many of us have used [How Old website](https://how-old.net/). It was a demo site from Microsoft to showcase Vision APIs as part of cognitive services. Vision API now can also describe the image and provide even more information about the image content such as genders of people in picture if any, classification of racial/adult content and so on. This is all coming from the booming of neural networks deep learning with breakthroughs in the last few years. Deep learning is more targeted for computer vision problems but can be applied to many other domains as well.

There are many tools to do deep learning ranging from cloud APIs like Microsoft cognitive service and Azure ML to open source frameworks like TensorFlow, Cafee, and Theano. In this post I will consume a ready-made TensorFlow model from a SharePoint addin. I will not develop & train the network from scratch as this is a time consuming process and TensorFlow community already published some models for common practical problems. TensorFlow is a bit hard and verbose and there are nice abstractions that make it more approachable and we still have the option to develop our custom models if needed. But for our case here I would like to stand on the shoulders of some giants and make use of something tested and trained.

So why do not we just use Azure ML or cognitive services vision API:


+ Maybe we have an architecture that is not allowed to call cloud services for security reasons or so.
+ The problem we are solving might need a custom model that is not available from providers like Microsoft & Google.
+ Some solutions need very fast response so models can be deployed and consumed directly on mobile phones for example.
+ An extra tiny benefit of using .NET on linux as we will see.

### Problem Description
I like SharePoint search feature for office/PDF documents. You can just dump all your documents and then use search to spot the information you need. For other types of documents you need to add more metadata to the lists/libraries hosting them so that you can find/order them. So our problem is to extend SharePoint picture library to store some metadata(description) for stored images. This can be ideally a SharePoint event receiver that will listen for new images created in the library and add the predicted description to them automatically. I will implement a slightly different solution. We will have a SharePoint addin that will add and extra button in the ribbon to open a popup window. In this window, the current image selected will be shown and y clicking a button we will call a .net core web API hosted in Ubuntu VM which will use bash to call TensorFlow model and come back with a few expected descriptions. The user will have the option to pick one of them and apply it to a description column in the current picture library.



### Design

High level design of this POC is to have a SharePoint addin calling ASP.NET core web API hosted in Ubuntu VM. The VM will have also TensorFlow installed and that ASP.NET web API will call TensorFlow to predict the image expected description. The POC is very simplistic as we will not secure the web API or use TensorFlow Serving for doing predictions (more suitable for production scenarios). Final result can be used for on premises deployment or in Office 365. If used in Office 365, then we will need to have an Ubuntu VM somewhere like in Azure or AWS to make it accessible on the internet.


Enough chit-chat, let's get our hands dirty.

Prerequisites:

- Windows 10 64bit
- VirtualBox
- Vs 2015 with Update 3
- [.net core tools](https://go.microsoft.com/fwlink/?LinkID=827546) 
- [Office Developer tools](https://www.visualstudio.com/vs/office-tools/)
- SharePoint team site (onprem 2013/2016 or simply Office 365 trial/dev tenant)


### 1- Create VM to host API & TensorFlow

Download Ubuntu 16.04 **64bit** and install it in a new VM using VirtualBox or your prefered VM tool.There will be some steps later about port forwarding and stuff like that so I prefer if we can stick to VirtualBox to follow same steps. Also I will work in a hybrid approach manner currently. Meaning, I will have the VM locally on my laptop (not in Azure) but will work with an Office 365 SharePoint site as I will be browsing the site locally on my laptop then the browser can do XHR calls to my VM. Also, remember to give the VM enough CPU/memory power as the default settings with VirtualBox starts with single CPU and things like that.To make the below steps easy to use, name the machine tensorflow both in VirtualBox and in Ubunto setup.

Once Ubuntu is installed, insert Guest Additions CD from Device menu of VirtualBox. You do not have to have any CD, VirtualBox will inject some ISO file and you will get a propmt to install VM guest additions. Restart the VM after this installation.

![install-guest-additions](/images/2017-01-26/install-guest-additions.png)

### 2 - Install TensorFlow inside Ubuntu VM

Open a bash terminal window and run the below commands:

```
$ sudo apt-get install python-pip python-dev
$ pip install tensorflow
```

Once the above is complete, you can test TensorFlow installation by running the below script in bash terminal :

```
$ python
>>> import tensorflow as tf
>>> a = tf.constant(10)
>>> b = tf.constant(32)
>>> sess = tf.Session() 
>>> print(sess.run(a + b))
>>> quit()

```

You should get something like:

![tensorflow-verify](/images/2017-01-26/tensorflow-verify.png)


### 3 - Install .NET Core

In a new bash terminal run the below commands to install .net core on ubuntu. The full instructions are documented [here](https://www.microsoft.com/net/core#linuxubuntu)


```
$ sudo sh -c 'echo "deb [arch=amd64] https://apt-mo.trafficmanager.net/repos/dotnet-release/ xenial main" > /etc/apt/sources.list.d/dotnetdev.list'
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 417A0893
$ sudo apt-get update
$ sudo apt-get install dotnet-dev-1.0.0-preview2.1-003177
```

Once installed you can also verify the sanity of the installation by running:

```
$ dotnet --version
```
![dotnet-core-installed](/images/2017-01-26/dotnet-core-installed.png)


### 4 - Clone and build VS solution

I have shared a Visual Studio solution containing the following projects:

- A SharePoint addin that acts as a front end to consume Web API 
- A .net core Web API project to call TensorFlow to call into the image 2 text model.

First clone that repo locally on the Windows machine (host) as we will need to build the solution

```
git clone https://github.com/ylashin/WhatsInsideImage.git
```

Once cloned, open solution file `WhatsInsideImage.sln` using Visual Studio but make sure to open Visual Studio as administrator.

Then open a command prompt and move to the Web API project to run the below:

```
dotnet restore
dotnet build -r ubuntu.16.04-x64
dotnet publish -c release -r ubuntu.16.04-x64
```

This will build and publish a standalone copy of the web API project that can be run on Ubuntu.

![dotnet-core-publish.png](/images/2017-01-26/dotnet-core-publish.png)

We need to copy the contents of the above highlighted publish folder inside the VM. Another way to simplify that in case we are doing lots of code changes and do not want to do too manual steps is to map that publish folder to a shared folder that can be accessed inside the VM.

So I will first open VM settings in VirtualBox and add a shared folder named `publish` to the target publish folder.

![shared-folder.png](/images/2017-01-26/shared-folder.png)

Inside the virtual machine open a terminal and run the below to mount this share to a local folder inside the VM

```
mkdir ~/publish
sudo mount -t vboxsf publish ~/publish

```
The mount command might need to be run every time you restart the VM. With the above you should have a local folder *~/publish* that contains published .net core files for our web API, so let us test it and verfiy some hello world thing first.


![share-folder.png](/images/2017-01-26/share-folder.png)

From a terminal window run the below to fire up our web API which runs in Kestrel:

```
$ cd publish
$ dotnet exec ./WhatsInsideImageApi.dll
```

![web-api-running.png](/images/2017-01-26/web-api-running.png)
The console shows that the aplication is running and can be accessed on `http://localhost:5000`. So fireup a browser and put `http://localhost:5000/api/describe` in the address bar to verify that the basic infrastructure works fine.

This is just a test endpoint that will echo current working direcotry plus current date

```
// GET api/describe
[HttpGet]
public IEnumerable<string> Get()
{
    var webRootPath = _hostingEnvironment.ContentRootPath;
    return new[] { webRootPath, DateTime.Now.ToString(CultureInfo.InvariantCulture) };
}
```
The expected output should be:

![test-endpoint.png](/images/2017-01-26/test-endpoint.png)


### 5 - Expose Web API to host machine

Our next steps is to make this API accessible outside the guest VM. We might think of just disabling the firewall inside Ubuntu and we should be good to go, but in most practical cases no one would be using Kestrel only as the web server. The defacto standard here is to have ngnix and let it proxy our API in Kestrel.

Inside the VM, open our friendly terminal to install nginx:

```
$ sudo apt-get install nginx
$ sudo service nginx start
```


We will now configure Nginx as a reverse proxy to forward requests to our ASP.NET application. We will be editing files in /etc folder so we need to run in admin mode 

```
sudo gedit /etc/nginx/sites-available/default
``` 
This will open nginx config file for default website in gedit editor, then we need to replace the **server** node contents with the following. 
The 0.0.0.0 means it will listen on any IP or network interface of the VM.

```
server {
    listen 0.0.0.0:5001;
    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```


![nginx-config.png](/images/2017-01-26/nginx-config.png)

Save the file and close the editor then from the terminal run the below to verify config and reload nginx:

```
$ sudo nginx -t
$ sudo nginx -s reload
```
Now if you hit the browser with `http://localhost:5001/api/describe` you should get the same result but this time through nginx.

Then to access the same API from host machine we need to disable Ubuntu firewall as below:
```
$ sudo ufw disable
```

Also VirtualBox VMs are created with NAT network adapter type by default in which the host cannot access the guest unless port forwarding is configured. So, from VM settings dialog switch to the network tab and then click port forwarding button to create the below rule. When you click OK, it might trigger some Windows Firewall dialog to ask you to open ports needed on VirtualBox network adapter. You should allow this access for sure to have things running.


Now if you hit the browser (in your host machine) with `http://localhost:5001/api/describe` you should get the same result but this time through port forwarding --> nginx --> Kestrel.

If you do not like port forwarding , you can configure the network of that VM to be bridged and this way you can just grab the IP of the VM and hit it remotely.
This has a downside of the IP might be different across diffrent sessions on the VM.

By default most web servers like IIS & nginx have default configuration with some settings to limit the size of payload they receive with HTTP requests. For our case we need to relax this setting a bit as we will be hitting web API with image binary contents.


In order to fix this issue, we need to edit nginx.conf file similar to waht we did with the default site config file.

```
$ sudo gedit /etc/nginx/nginx.conf
```
Search for this variable: **client_max_body_size**. If you find it, just incrmodify its value to 10M, for example. If it doesn’t exist, then you can add it inside and at the end of **http** { … } block.

`client_max_body_size 20M;`

Save the file and close the editor then from the terminal run the below to verify config and reload nginx:

```
$ sudo nginx -t
$ sudo nginx -s reload
```


### 6 : Install prerequisites for running IM2TXT model

First let us have some history background about TensorFlow, the below from Wikipedia shows what TensorFlow is :

> TensorFlow is an open source software library for machine learning in various kinds of perceptual and language understanding tasks. It is currently used for both research and production by 50 different teams in dozens of commercial Google products, such as speech recognition, Gmail, Google Photos, and search, many of which had previously used its predecessor DistBelief. TensorFlow was originally developed by the Google Brain team for Google's research and production purposes and later released under the Apache 2.0 open source license on November 9, 2015.


As I mentioned above, Google and the community have shared some useful models on a github repo called [TensorFlow Models](https://github.com/tensorflow/models).
One of those models is called im2txt which takes an image as input and returns a few expected descriptions as output, simple enough!

Unfortunately those shared models are untrained neural network definition. Some nice guys volunteered to do the training  and share the final model with the tuned parameters, you can find more details about that in : [Hints about how to use pretrained models](https://github.com/tensorflow/models/issues/466)

So we wil just use some pretrained model from that issue #466 plus the original instructions from [im2txt page](https://github.com/tensorflow/models/tree/master/im2txt) to first prepare the model and run it from the shell before consuming it from web API.

We need to install JDK as it is required for Bazel which is Google build tool, so again the the terminal we run:
	
```
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
$ java -version 
```

The above should show some java version of 1.8.x

Then next is Bazel installation.

```
$ echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
$ curl https://bazel.build/bazel-release.pub.gpg | sudo apt-key add -
$ sudo apt-get update && sudo apt-get install bazel
$ sudo apt-get upgrade bazel
```

Now if you run `$ bazel`, you should get some details and version info 

![bazel.png](/images/2017-01-26/bazel.png)


Next we need to install numpy & NLTK (probably they are needed for running the training not the prediction)
```
$ sudo apt-get install python-numpy
$ sudo pip install -U nltk
```


Install git to allow us to clone TensorFlow models repo
```
$ cd ~
$ sudo apt-get install git-core
$ git --version
$ git clone https://github.com/tensorflow/models.git
$ cd models
$ git reset --hard 9997b250
```

The last line is to reset our repo HEAD to a certain commit as there are some recent breaking changes in TensorFlow & Models repos that will cause some stuff not to work.


### 7: Prepare TensorFlow model for prediction
In this step we will be following the [Generating Captions](https://github.com/tensorflow/models/tree/master/im2txt#generating-captions) section of the im2txt page and also the comment by siavashk on 16/11/2016 for issue #466.

So we will start by all files to be used for the prediction Final files are stored on Google Drive as per siavashk comment but I also uploaded them to code repo related with this post to be able to grab them easily without the risk of files removed from Google Drive or something. I also updated the vocabulary text file to remove some unneeded quotes as per the [comment from @cshallue on Oct 16, 2016](https://github.com/tensorflow/models/issues/466). You do not need to do anything of the above all files are shared and ready to be used directly.

```
$ cd ~
$ mkdir model-data
$ cd model-data
$ wget graph.pbtxt
$ wget model.ckpt-2000000
$ wget model.ckpt-2000000.meta
$ wget word_counts.txt
$ wget http://www.horsebreedsinfo.com/images/fast_horse_riding.jpg

```

The above just downloads model trained data plus one sample test image we will be using in the next step.
Then the next snippet will just compile some inference tool that can be called using a shell command to do the predition.

```
$ cd ~/models/im2txt
$ bazel build -c opt im2txt/run_inference
$ export CUDA_VISIBLE_DEVICES=""
$ bazel-bin/im2txt/run_inference --checkpoint_path="/home/super/model-data/model.ckpt-2000000" --vocab_file="/home/super/model-data/word_counts.txt" --input_files="/home/super/model-data/fast_horse_riding.jpg"
```
We should see something like :

![sample-predition.png](/images/2017-01-26/sample-predition.png)





### Other ideas/ Troubleshooting tips

- The solution implemented is very simplistic to have something running quickly. Actually for TensorFlow, the production way of doing predictions is to use something called TensorFlow Serving but this would be too much for the first adventure. Also we can use SharePoint remote event receivers to automate the process or maybe allow the user to edit the description to correct/enrich it.
- After deplying Web API app, the .sh file might need to be updated with run permissions.
- To test from SharePoint picture library page we need to open chrome with mixed content allowed otherwise we will need to configure SSL with nginx.
Chrome can be opened to allow mixed content by running:
`"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe" --allow-running-insecure-content`
- Web API project has also to be configured to allow CORS calls and currently it is accepting all domains, have a look on source code if you would like to limit it to certain domains. Without this CORS configuration, AJAX calls from browsers/user agents will not be able to access it.
- When testing keep fiddler running to check stuff like CORS/Firewall Access/Mixed Content/Network calls

### Resources

- [Deep Learning free course by Google](https://www.udacity.com/course/deep-learning--ud730)
- [im2txt model](https://github.com/tensorflow/models/tree/master/im2txt)
- [Publish .net code to a Linux Production Environment](https://docs.microsoft.com/en-us/aspnet/core/publishing/linuxproduction)