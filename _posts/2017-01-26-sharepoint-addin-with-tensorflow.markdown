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
- [Hints about how to use pretrained models](https://github.com/tensorflow/models/issues/466)
- 