---
title: Getting Started - FeedHenry Hello World
banner: post-bg.png
permalink: getting_started__feedhenry_hello_world
date: 2015-05-07 00:00:00
tags:
- feedhenry
---

This guide demonstrates step-by-step how to create and run your first FeedHenry Hello World project.
<!-- more -->

# Step 1: Create A New FeedHenry Project

When you log into your FeedHenry domain, you should end up at a landing page like the one pictured below.

{% asset_img "landing_page.png" [Landing Page] %}

Click on the "__Projects__" panel to go to the projects page.

{% asset_img "projects_panel.png" [Projects Panel] %}

You should see all of your existing projects (if you have any). Click on the "__New Project__" button to create a new project.

{% asset_img "new_project_button.png" [New Project Button] %}

You should see the "__Hello World Project__" template toward the top of the list. If you don't, use the search box to find it.

{% asset_img "hello_world_project.png" [Hello World Project] %}

Click the "__Choose__" button to the right to select the template.

{% asset_img "hello_world_project_choose_button.png" [Hello World Project] %}

Fill in the project name and click the "__Create__" button.

{% asset_img "hello_world_project_options.png" [Hello World Project] %}

Your project should begin the creation process. This can take a few minutes and you will be presented with a progress bar and status text box.

{% asset_img "hello_world_project_status_in_progress.png" [Hello World Project] %}

When your project has been successfully created, you will see the following _successful_ status messages.

{% asset_img "hello_world_project_status_complete.png" [Hello World Project] %}

Your project is now created and ready to use.

# Step 2: Explore Your Project

Now that you have a project, let's explore the structure a bit. You should see a project landing page similar to the one below.

{% asset_img "explore_project.png" [Explore Project] %}

Click on the "__Cloud App__" panel to explore the backend Hello World cloud app.

{% asset_img "explore_project_cloud_code.png" [Explore Project] %}

You should see the "__Details__" page for the cloud app. From here you can stop/start the cloud app as well as see other various bits of information such as the host it's currently running on and which environment it's running in.

{% asset_img "explore_project_cloud_code_details.png" [Explore Project] %}

On the left column, click on the "__Editor__" button.

{% asset_img "explore_project_cloud_code_editor_button.png" [Explore Project] %}

You should now see the directory structure of the source code for the cloud app. Open up the "`/lib/hello.js`" file to view the source that actually does the work for this Hello World project. While you're at it, click around on some of the other files and take a look at their contents to get an idea of how everything is glued together.

{% asset_img "explore_project_cloud_code_editor.png" [Explore Project] %}

Now go back to the project's landing page and click on the "__Cordova Light App__" panel to explore the mobile app.

{% asset_img "explore_project_apps.png" [Explore Project] %}

You should see the "__Details__" page for the mobile app.

{% asset_img "explore_project_apps_details.png" [Explore Project] %}

From there, you can test the mobile app directly from the web browser. If you'd like to test directly from your phone, you can go to the "__Export__" page and export/download a copy of the app for your desired platform.

# Step 3: Test Your Project

Now that you have a project, and you've explored the structure a bit, let's run a quick test using the browser based device simulator. Enter your name into the input box and click "__Say Hello From The Cloud__".

{% asset_img "test_project.png" [Test Project] %}

You should see results similar to those pictured below.

{% asset_img "test_project_result.png" [Test Project] %}

That's it! You've successfully created and tested a Hello World app in FeedHenry.
