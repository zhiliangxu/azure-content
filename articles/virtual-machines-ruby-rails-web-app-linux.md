<properties 
	pageTitle="Ruby on Rails Web App on Azure using Linux VM" 
	description="Host a Ruby on Rails-based website on Azure using a Linux virtual machine." 
	services="virtual-machines" 
	documentationCenter="ruby" 
	authors="blackmist" 
	manager="wpickett" 
	editor=""/>

<tags 
	ms.service="virtual-machines" 
	ms.workload="web" 
	ms.tgt_pltfrm="vm-linux" 
	ms.devlang="ruby" 
	ms.topic="article" 
	ms.date="09/17/2014" 
	ms.author="larryfr"/>





#Ruby on Rails Web application on an Azure VM

This tutorial describes how to host a Ruby on Rails-based website on Azure using a Linux virtual machine. This tutorial assumes you have no prior experience using Azure. Upon completing this tutorial, you will have a Ruby on Rails-based application up and running in the cloud.

You will learn how to:

* Setup your development environment

* Setup an Azure virtual machine to host Ruby on Rails.

* Create a new Rails application

* Copy files to the virtual machine using scp 

The following is a screenshot of the completed application:

![a browser displaying Listing Posts][blog-rails-cloud]

##In this article

* [Set up your development environment](#setup)

* [Create a Rails application](#create)

* [Test the application](#test)

* [Create an Azure Virtual Machine](#createvm)

* [Copy the application to the VM](#copy)

* [Install gems and start the application](#start)

* [Next steps](#next)

##<a id="setup"></a>Set up your development environment

1. Install Ruby in your development environment. Depending on your operating system, the steps may vary.

	* **Apple OS X** - There are several Ruby distributions for OS X. This tutorial was validated on OS X by using [Homebrew](http://brew.sh/) to install **rbenv** and **ruby-build**. Installation information can be found at [https://github.com/sstephenson/rbenv/](https://github.com/sstephenson/rbenv/).

	* **Linux** - Use your distributions package management system. This tutorial was validated on Ubuntu 12.10 using the ruby1.9.1 and ruby1.9.1-dev packages.

	* **Windows** - There are several Ruby distributions for Windows. This tutorial was validated using [RailsInstaller](http://railsinstaller.org/) 1.9.3-p392.

2. Open a new command-line or terminal session and enter the following command to install Ruby on Rails:

		gem install rails --no-rdoc --no-ri

	> [AZURE.NOTE] This command may require administrator or root privileges on some operating systems. If you receive an error while running this command, try using 'sudo' as follows:
	>
	>````` 
	sudo gem install rails
	`````

	> [AZURE.NOTE] Version 3.2.12 of the Rails gem was used for this tutorial.

3. You must also install a JavaScript interpreter, which will be used by Rails to compile CoffeeScript assets used by your Rails application. A list of supported interpreters is available at [https://github.com/sstephenson/execjs#readme](https://github.com/sstephenson/execjs#readme).
	
	[Node.js](http://nodejs.org/) was used during validation of this tutorial, as it is available for OS X, Linux and Windows operating systems.

##<a id="create"></a>Create a Rails application

1. From the command-line or terminal session, create a new Rails application named "blog_app" by using the following command:

		rails new blog_app

	This command creates a new directory named **blog_app**, and populates it with the files and sub-directories required by a Rails application.

	> [AZURE.NOTE] This command may take a minute or longer to complete. It performs a silent installation of the gems required for a default application, and during this time may appear to hang.

2. Change directories to the the **blog_app** directory, and then use the following command to create a basic blog scaffolding:

		rails generate scaffold Post name:string title:string content:text

	This will create the controller, view, model, and database migrations used to hold posts to the blog. Each post will have an author name, title for the post, and text content.

3. To create the database that will store the blog posts, use the following command:

		rake db:migrate

	This will use the default database provider for Rails, which is the [SQLite3 Database][sqlite3]. While you may wish to use a different database for a production application, SQLite is sufficient for the purposes of this tutorial.

##<a id="test"></a>Test the application

Perform the following steps to start the Rails server in your development environment

1. From the command-line or terminal session, start the rails server using the following command:

		rails s

	You should see output similar to the following. Note the port that the web server is listening on. In the example below, it is listening on port 3000.

		=> Booting WEBrick
		=> Rails 3.2.12 application starting in development on http://0.0.0.0:3000
		=> Call with -d to detach
		=> Ctrl-C to shutdown server
		[2013-03-12 19:11:31] INFO  WEBrick 1.3.1
		[2013-03-12 19:11:31] INFO  ruby 1.9.3 (2012-04-20) [x86_64-linux]
		[2013-03-12 19:11:31] INFO  WEBrick::HTTPServer#start: pid=9789 port=3000

2. Open your browser and navigate to http://localhost:3000/. You should see a page similar to the following:

	![default rails page][default-rails]

	This page is a static welcome page. To see the forms generated by the scaffolding command, navigate to http://localhost:3000/posts. You should see a page similar to the following:

	![a page listing posts][blog-rails]

	To stop the server process, enter CTRL+C in the command-line

##<a id="createvm"></a>Create an Azure Virtual Machine

Follow the instructions given [here][vm-instructions] to create an Azure virtual machine that hosts Linux.

> [AZURE.NOTE] the steps in this tutorial were performed on an Azure Virtual Machine hosting Ubuntu 12.10. If you are using a different Linux distribution, different steps may be required to accomplish the same tasks.

> [AZURE.IMPORTANT] You **only** need to create the virtual machine. Stop after learning how to connect to the virtual machine using SSH.

After creating the Azure Virtual Machine, perform the following steps to install Ruby and Rails on the virtual machine:

1. From the command-line or terminal session, use the following command to connect to the virtual machine using SSH:

		ssh username@vmdns -p port

	Substitute the user name specified during the creation of the VM, the DNS address of the VM, and the port of the SSH endpoint. For example:

		ssh railsdev@railsvm.cloudapp.net -p 61830

	> [AZURE.NOTE] If you are using Windows as your development environment, you can use a utility such as **PuTTY** for SSH functionality. PuTTY can be obtained from the [PuTTY download page](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html).

2. From the SSH session, use the following commands to install Ruby on the VM:

		sudo apt-get update -y
		sudo apt-get upgrade -y
		sudo apt-get install ruby1.9.1 ruby1.9.1-dev build-essential libsqlite3-dev nodejs -y

	After the installation completes, use the following command to verify that Ruby has been successfully installed:

		ruby -v

	This should return the version of Ruby that is installed on the virtual machine, which may be different than 1.9.1. For example, **ruby 1.9.3p194 (2012-04-20 revision 35410) [x86_64-linux]**.

2. Use the following command to install Bundler:

		sudo gem install bundler --no-rdoc --no-ri

	Bundler will be used to install the gems required by your rails application once it has been copied to the server.

##<a id="copy"></a>Copy the application to the VM

From your development envirionment, open a new command-line or terminal session and use the **scp** command to copy the **blog_app** directory to the virtual machine. The format for this command is as follows:

	scp -r -P 54822 -C directory-to-copy user@vmdns:

For example:

	scp -r -P 54822 -C ~/blog_app railsdev@railsvm.cloudapp.net:

> [AZURE.NOTE] If you are using Windows as your development environment, you can use a utility such as **pscp** for scp functionality. Pscp can be obtained from the [PuTTY download page](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html).

The parameters used for this command have the following effect:

* **-r**: Recursively copies the contents of the specified directory and all sub-directories

* **-P**: Use the specified port for SSH communication

* **-C**: Enable compression

* **directory-to-copy**: The local directory to be copied

* **user@vmdns**: The address of the machine to copy the files to, as well as the user account to log in with

After the copy operation, the **blog_app** directory will be located in the users home directory. Use the following commands in the SSH session with the virtual machine to view the files that were copied:

	cd ~/blog_app
	ls

The list of files returned should match the files contained in the **blog_app** directory in your development environment.

##<a id="start"></a>Install gems and start Rails

1. On the virtual machine, change directories to the **blog_app** directory and use the following command to install the gems specified in the **Gemfile**:

		sudo bundle install

2. To create the database, use the following command:

		rake db:migrate

3. Use the following command to start the server:
	
		rails s

	You should see output similar to the following. Note the port that the web server is listening on. In the example below, it is listening on port 3000.

		=> Booting WEBrick
		=> Rails 3.2.12 application starting in development on http://0.0.0.0:3000
		=> Call with -d to detach
		=> Ctrl-C to shutdown server
		[2013-03-12 19:11:31] INFO  WEBrick 1.3.1
		[2013-03-12 19:11:31] INFO  ruby 1.9.3 (2012-04-20) [x86_64-linux]
		[2013-03-12 19:11:31] INFO  WEBrick::HTTPServer#start: pid=9789 port=3000

2. In your browser, navigate to the [Azure Management Portal][management-portal] and select your Virtual Machine.

	![virtual machine list][vmlist]

3. Select **ENDPOINTS** at the top of the page, and then click **+ ADD ENDPOINT** at the bottom of the page.

	![endpoints page][endpoints]

4. In the **ADD ENDPOINT** dialog, click the arrow in the bottom left to continue to the second page, and enter the following information in the form:

	* **NAME**: HTTP

	* **PROTOCOL**: TCP

	* **PUBLIC PORT**: 80

	* **PRIVATE PORT**: &lt;port information from step 3 above&gt;

	This will create a public port of 80 that will route traffic to the private port of 3000 - where the Rails server is listening.

	![new endpoint dialog][new-endpoint]

5. Click the checkmark to save the endpoint.

6. A message should appear that states **UPDATE IN PROGRESS**. Once this message disappears, the endpoint is active. You may now test your application by navigating to the DNS name of your virtual machine. The website should appear similar to the following:

	![default rails page][default-rails-cloud]

	Appending **/posts** to the URL should display the pages generated by the scaffolding command.

	![posts page][blog-rails-cloud]

##<a id="next"></a>Next steps

In this article you have learned how to create and publish a basic forms-based Rails application to an Azure Virtual Machine. Most of the actions we performed were manual, and in a production environment it would be desirable to automate. Also, most production environments host the Rails application in conjunction with another server process such as Apache or NginX, which handles request routing to multiple instances of the Rails application and serving static resources.

For information on automating deployment of your Rails application, as well as using the Unicorn web server and NginX, see [Unicorn+NginX+Capistrano with an Azure Virtual Machine][unicorn-nginx-capistrano].

If you would like to learn more about Ruby on Rails, visit the [Ruby on Rails Guides][rails-guides].

To learn how to use the Azure SDK for Ruby to access Azure services from your Ruby application, see:

* [Store unstructured data using blobs][blobs]

* [Store key/value pairs using tables][tables]

* [Serve high bandwidth content with the Content Delivery Network][cdn-howto]



<!-- WA.com links -->
[blobs]: /en-us/documentation/articles/storage-ruby-how-to-use-blob-storage

[cdn-howto]: /en-us/develop/ruby/app-services/

[management-portal]: https://manage.windowsazure.com/

[tables]: /en-us/develop/ruby/how-to-guides/table-service/

[unicorn-nginx-capistrano]: /en-us/documentation/articles/virtual-machines-ruby-deploy-capistrano-host-nginx-unicorn/

[vm-instructions]: /en-us/documentation/articles/virtual-machines-linux-tutorial


<!-- External Links -->
[rails-guides]: http://guides.rubyonrails.org/

[sqlite3]: http://www.sqlite.org/

<!-- Images -->
[blog-rails]: ./media/virtual-machines-ruby-rails-web-app-linux/blograilslocal.png

[blog-rails-cloud]: ./media/virtual-machines-ruby-rails-web-app-linux/blograilscloud.png 

[default-rails]: ./media/virtual-machines-ruby-rails-web-app-linux/basicrailslocal.png

[default-rails-cloud]: ./media/virtual-machines-ruby-rails-web-app-linux/basicrailscloud.png

[vmlist]: ./media/virtual-machines-ruby-rails-web-app-linux/vmlist.png

[endpoints]: ./media/virtual-machines-ruby-rails-web-app-linux/endpoints.png

[new-endpoint]: ./media/virtual-machines-ruby-rails-web-app-linux/newendpoint.png

