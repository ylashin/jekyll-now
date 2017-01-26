---
title:  "SharePoint meets TensorFlow!"
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



#### Design

High level design of this POC is to have a SharePoint addin calling ASP.NET core web API hosted in Ubuntu VM. The VM will have also TensorFlow installed and that ASP.NET web API will call TensorFlow to predict the image expected description. The POC is very simplistic as we will not secure the web API or use TensorFlow Serving for doing predictions (more suitable for production scenarios). Final result can be used for on premises deployment or in Office 365. If used in Office 365, then we will need to have an Ubuntu VM somewhere like in Azure or AWS to make it accessible on the internet.


Enough chit-chat, let's get our hands dirty.

PreReq:
* Windows 10 64bit
* VirtualBox
* Vs 2015 with Update3 + [.net core tools](https://go.microsoft.com/fwlink/?LinkID=827546) + [Office Developer tools](https://www.visualstudio.com/vs/office-tools/)
* SharePoint team site (onprem 2013/2016 or simply Office 365 trial/dev tenant)


### 1- Create VM to host API & TensorFlow

Download Ubuntu 16.04 **64bit** and install it in a new VM using VirtualBox or your prefered VM tool.There will be some steps later about port forwarding and stuff like that so I prefer if we can stick to VirtualBox to follow same steps. Also I will work in a hybrid approach manner currently. Meaning, I will have the VM locally on my laptop (not in Azure) but will work with an Office 365 SharePoint site as I will be browsing the site locally on my laptop then the browser can do XHR calls to my VM. Also, remember to give the VM enough CPU/memory power as the default settings with VirtualBox starts with single CPU and things like that.To make the below steps easy to use, name the machine tensorflow both in VirtualBox and in Ubunto setup.

Once Ubuntu is installed, insert Guest Additions CD from Device menu of VirtualBox.
You do not have to have any CD, VirtualBox will inject some ISO file and you will get a propmt to install VM guest additions.
Restart the VM after this installation.

Then we will need to create a Share named my-share in VM settings and register it in
the VM as below pointing to a local folder inside the VM also.
We will use this folder to copy published files for .net core web API.
To simplify development and testing, The share folder on the host was actually the publish target folder of .NET web API application.

mkdir ~/host-share
sudo mount -t vboxsf my-share ~/host-share # this might need to be registered to run with every restart




### Other ideas
The solution implemented is very simplistic to have something running quickly. Actually for TensorFlow, the production way of doing predictions is to use something called TensorFlow Serving but this would be too much for the first adventure. Also we can use SharePoint remote event receivers to automate the process or maybe allow the user to edit the description to correct/enrich it.