---
title: Contribute a Task
category: Contributing
order: 2
---

Along with contributing documentation and general code, the cool thing about watchme
is that it's fairly easy to add a new kind of watcher task! A task can literally
do anything that your imagination can come up, within the constraints of your
local machine, ranging from interaction with web APIs to system resources and 
networking. Follow this guide for instructions.

## 1. Add a Task Folder

First, create a new folder under `watchme/tasks` that corresponds with the name of your
task. For example, the urls watcher is located at `watchme/tasks/urls`. 
Actually, it's usually easiest to copy an entire watcher folder as a template to start:

```bash
cp -R watchme/watchers/urls watchme/watchers/mytype
```

## 2. Task Setup

You will then need to add your new task name and or links to the following files:

### List of Watchers
There is a [list of watchers](https://github.com/vsoch/watchme/blob/master/watchme/watchers/README.md) 
(bullets) in the watchers folder README.md. Add a relative link to your folder there. This is to
help those browsing GitHub to find the folder (and you can add some details about what kinds of tasks
are included)

### Dependencies
The [version.py](https://github.com/vsoch/watchme/blob/master/watchme/version.py) file defines
sets of dependencies for different "extra_install" options provided by watchme. For example,
for the urls task folder there is a "dynamic" function that uses Python's beautiful soup
to parse html. We add this to version.py:

```python
## beautiful soup selection task

INSTALL_URLS_DYNAMIC = (
    ('beautifulsoup4', {'min_version': '4.6.0'}),
)
```

Also add the new set of dependencies to the `INSTALL_ALL` variable (if the user runs
`pip install watchme[all]`:

```python
INSTALL_ALL = (INSTALL_REQUIRES +
               INSTALL_URLS_DYNAMIC)
```

And then in the [setup.py](https://github.com/vsoch/watchme/blob/master/setup.py) we add a named
extra install to point to it (the variable URLS_DYNAMIC):

```python
INSTALL_REQUIRES = get_requirements(lookup)
URLS_DYNAMIC = get_requirements(lookup,'INSTALL_URLS_DYNAMIC')
```

And add it to the list here:

```python
extras_require={
    'all': [INSTALL_REQUIRES],
    'urls-dynamic': [URLS_DYNAMIC]
},
...
```

And then the interested user would install the extra dependecies like:

```bash
$ pip install watchme[urls-dynamic]
```

You should be sure to write instructions for how to do this in the documentation for
your task folder (discussed later).

### List of Task Types

Add the task name to the `WATCHME_TASK_TYPES` in [defaults.py](https://github.com/vsoch/watchme/blob/master/watchme/defaults.py)

```python
WATCHME_TASK_TYPES = ['urls', 'url', 'mytype']
```

### Import of Task

Also in the `watchme/watchers/__init__.py` you should add a handler to import
the Task type from the correct folder:

```python
# Validate variables provided for task
if task_type.startswith('url'):
    from .urls import Task

# Validate variables provided for task
if task_type == 'psutils':
    from .psutils import Task
```

If you forget the handler, an error message will alert you when you try to add
a task from your set.

## 3. Add your Task

In the `__init__.py` folder you should put your watcher class called "Task" that instantiates the TaskBase
class as a parent:

```python
'''

Copyright (C) 2019 Vanessa Sochat.

This Source Code Form is subject to the terms of the
Mozilla Public License, v. 2.0. If a copy of the MPL was not distributed
with this file, You can obtain one at http://mozilla.org/MPL/2.0/.


'''
from watchme.tasks import TaskBase
from watchme.logger import bot
import os
import sys

class Task(TaskBase):

    required_params = ['url']

    def __init__(self, name, params=[], **kwargs): 

        self.type = 'urls'

        # Handles setting the name, setting params, and validate
        super(Task, self).__init__(name, params, **kwargs)

```

Under `self.type` you should put the name, which is usually the same as the folder
(urls shown above). If you have any required parameters (the minimum set for the
task to run) put them under `required_params`.


## 4. Write a validation function

The parent class is already going to check that the user has provided the `required_params`,
so you can implement an (optional) `_validate` function that does additional input
checks. This check will be performed both when the user adds a new task, and when 
a task is run, in the case that the user decided to manually edit a configuration 
file and invalidate a task.

```python
    def _validate(self):
        '''additional validation function, called by validate() of 
           superclass. Here we assume all required self.params are included.
           If an parameter is found to be invalid, self.valid should be set
           to False
        '''
        # The url must begin with http
        if not self.params['url'].startswith('http'):
            bot.error('%s is not a valid url.' % self.params['url'])
            self.valid = False
```

If one or more parameters are found to be invalid, you should set self.valid to False.
You don't need to exit from the function. You also don't need to have this function if no further validation is needed.

### 5. Write a function to run your task

The watcher client is going to be assembling the list of tasks to run, and then
running them. Specifically, it's going to be creating an instance of your Task
class, and handing it the entire (dictionary) of parameters as key value pairs:

```python
task = Task(params)
```

In the example above, params looks like this:

```python
{
    "url": "https://www.reddit.com/r/hpc",
    "active": "true",
    "type": "urls",
    "uri": "task-reddit-hpc"
}
```

The user generated this task at the command line only providing a url:

```bash
$ watchme add-task [watcher] task-reddit-hpc url@https://www.reddit.com/r/hpc
```

And the variables for active, the unique resource identifier (uri) and the
task type were added either as a default setting (type) or a default variable
set by the watcher (active and uri). If your watcher were called something
different (e.g., network) then the command would have looked like this:

```bash
$ watchme add-task [watcher] task-reddit-hpc url@https://www.reddit.com/r/hpc --type network
```

It follows that in the instantiation of your class, it must return a task object.
If the `task.valid` is True, you are good to go. If `task.valid` is False, the
task won't be run. 

### 6. Write Task Functions

The multiprocessing workers are going to expect, for each task, to be able
to export a set of parameters (dictionary of key value pairs, usually just
the task.params object) and a function to run. Thus, we use the following
functions:

```python
task.export_params()
```

This function is already implemented, and will return the task.params. 

```python
task.export_func()
```

This function is required to be implemented by your Task subclass. The function
should expect to take one or more keyword arguments. If your task type just
has one function, it's fairly straight forward to import and return the function. 
If you choose between one or more functions based on user variables, you can implement
that logic here, and return the correct one. Where should you store the task
functions? You can put them in a `tasks.py` located in the same folder:

```bash
watchme/
  watchers/
    urls/
      __init__.py
      tasks.py
```

#### Example Task

Here is a simple example task. Notice that the required argument url is a positional
argument, and the rest (anything could be passed from the configuration, potentially, including
optional args) are represented with kwargs.

```python
def get_task(url, **kwargs):
    '''a simple task to use requests to get a url. By default, we return
       the raw response.

       Parameters
       ==========
       url: a url to return the page for (required
    '''
    result = None
    response = requests.get(url)
    if response.status_code == 200:
        result = response.text
    return result
```

Notice also that if there is an issue retrieving the url, the result is returned
as None.


#### Rules for Tasks

The following rules should pertain to writing tasks:

  1. The input variables must coincide with the variables named by the task.params. The task should accept some exploded list of of **kwargs to be flexible to do this.
  2. The task should return some finished file object, string, or other text matter that the watcher can then update in the repository (see next section).
  3. If the function is not successful, return None.


#### Return Values

The return value of your task is going to determine how it's processed by the watcher.

##### File Objects

If your function returns a file that is found to exist, it will be moved into the task folder.
If your function names the file, it will be moved without changing the name. If the user defined
a `file_name` parameter, it will be renamed to this. If you don't do a good job to name your
file and the user doesn't specify this parameter, it will be the basename of the url provided.

##### Json Objects

If you want to write a json to file, either return a path to file (somewhere in tmp to be
moved to the repository) or return a dictionary type, and it will be written to `result.json`
in the task folder. If your user (or default task) specifies a `file_name` variable, it will
be named this instead of `result.json`

##### Text

If a string is provided and it doesn't exist as a path, it's assumed to be some text to write to file.
It will by default written to `result.txt` unless another `file_name` parameter is specified.


##### Lists

If your task returns a list, watchme will do it's best to attempt to sniff the content,
and figure out what kind of save you want. To make things simply, you are allowed to
return lists, but all of the content must be of the same save type.

 1. If the list is empty, no further action is taken.
 2. If the list is provided with save_as@json, then the entire list is saved as a single json object.
 3. If the list is provided and save_as@json_list is set, each object in the list (should be json or dict) is saved as a separate json object.
 4. If the first item in the list is a path that exists, the entire list is assumed to be files that should be copied to the repository.
 5. Otherwise, each item in the list is saved as text content.

As before, if you want to have a custom name for the items in the list, either write the files yourselves (and they will
be copied) or set the `file_name@custom.txt` variable. If you set a custom name, the list of items will be named like:

```bash
watcher/
    task-name/
        custom0.txt
        custom1.txt
        ...
        customN.txt
```

If there is some functionality you aren't able to achieve with return types, or you
would like a new return type added, please [open an issue]({{ site.repo }}/issues).

#### Variables for Tasks

You should tell your users (in the task function header, and documentation for it)
what variables are allowed to be set for the task. If the variable is defined
in the task.kwargs (from the watcher configuration) your function can check for it,
and return a default. For example, let's say I wanted to give my users the optional
to disable ssl checking when downloading an object. I would tell them to set
`disable_ssl_check` when they create the task:

```bash
$ watchme add-task [watcher] task-dangerous url@https://www.download/big/thing disable_ssl_check@true
```

And then my function could check for it, and set a default.

```python
verify = True
if "disable_ssl_check" in kwargs:
    verify = False
```

As another example, let's say the task will by default write the content to a file.
If you wanted the user to be able to specify writing binary, you could tell them about
a `write_format` variable:

```python
write_format = kwargs.get('write_format', 'w')
```

It would always default to "w" unless otherwise specified:

```bash
$ watchme add-task [watcher] task-download-binary url@https://www.download/big/thing write_format@wb
```

### 5. Write Documentation

Finally, you should write up all of the usage examples and variables into 
a documentation file for the watcher! These are located under `docs/_docs/watcher-tasks` 
and the file should be named according to the task type. For example, the "urls"
task shown above has a file named `docs/_docs/watcher-tasks/urls.md`. In the front
end matter, you should only need to change the title, and the permalink for your 
type (it should be in the format `/watchers/<name>`:

```
---
title: URLS
category: Watcher Tasks
permalink: /watchers/urls/
order: 2
---
```

Then in the index.md in that same folder, add a link for your watcher page:

```
 - [url watcher]({{ site.baseurl }}/watchers/urls/) to watch for changes in web content
```

Write as much detail in the documentation as you think necessary. Generally, you want
to say:

 - required and optional parameters
 - usage examples
 - functions available for the task, and how to specify them using the parameter "func"
