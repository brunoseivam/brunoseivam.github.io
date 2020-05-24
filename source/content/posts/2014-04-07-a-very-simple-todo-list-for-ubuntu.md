---
title: A very simple ToDo Indicator list for Ubuntu
author: Bruno Martins
date: 2014-04-07T20:06:31+00:00
url: /a-very-simple-todo-list-for-ubuntu/
dsq_thread_id:
  - 2701954890
categories:
  - linux
  - python

---
![Screenshot of the running app](img/todo-appindicator.jpg)

Pretty, huh? The icon is from the [Open Icon Library](http://openiconlibrary.sourceforge.net/gallery2/?./Icons/actions/dialog-ok-apply-4.png "Yay! Icons!").

(You can synchronize your ToDo list over multiple PC's just placing the database in a Dropbox folder)

# The Motivation

I have to admit: I'm not an organized person. Over the past few months I have kind of forced myself to write down everything I was doing and all the thoughts I was having, for future (and present) reference. A natural pattern that emerged was that I always had a ToDo list on one of the margins of the most recent notebook page. As I wrote more things on the notebook, I ended up copying the ToDo list to a more recent page.

So I decided to build a simple and easily accessible ToDo list.

I wanted the app to be easily accessible and not get in the way of other things. Therefore, I decided to build it as an Ubuntu Application Indicator (those pesky icons that stand next to the system clock).

I wrote this app using Python; the code is available on [GitHub](https://github.com/brunoseivam/todoindicator "ToDo Indicator on Github!").

# The Specs

Ideally I wanted to have a more dynamic Indicator, allowing me to add, remove and check items using the dropdown menu directly. However, the appindicator library for Python doesn't allow me to do that, as I can't embed arbitrary GTK Widgets inside a GTK MenuItem.

So, the menu was designed as follows:

  * A list of items. Each item has an indication of Done/Not Done (Unicode [checkboxes](http://en.wikipedia.org/wiki/Checkbox#Unicode "Unicode checkboxes ☑")). Clicking the item toggles its Done/Not Done status;
  * A button that allows the list of items to be changed. This button launches GEdit to edit the database;
  * A button to close the app.

# The Database

The database is composed of a single text file in the following format:

  * Each line contains one item;
  * The line starts with the string "Yes" or the string "No", representing the Done/Not Done state;
  * A semicolon follows;
  * A description of the item follows.

For example, the file below quite obviously describes two (easily accomplishable) tasks: the first one marked as Done and the second one as Not Done.

    Yes;Write a blog post about my ToDo list app
    No;Get obnoxiously rich

# The Code

The code relies on three libraries: [pygtk](http://www.pygtk.org/ "PyGTK"), [appindicator](https://unity.ubuntu.com/projects/appindicators/ "AppIndicators") (with faulty documentation) and [pyinotify](http://pyinotify.sourceforge.net/ "PyINotify"). The first one provides an API to build the menu. The second one provides bindings to attach the menu to the system as an Indicator. The last one allows watching for changes in the "database".

When the application starts it loads the "database" file and fills the menu. If the database changes, the app reloads it and rebuild the menu. If an item is clicked, its state is toggled and the database is updated (which in turn triggers a reload).

