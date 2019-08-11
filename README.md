# Welcome to my website project
This is the project that will serve as container of all my articles to document my projects and ideas..

## How to setup and run
Please follow the following steps

1. Setup or update the Gems
Be sure you have installed Ruby in your system.
If this is the first time you execute the project run:

```
gem install
```
But if you have added new gems or changed somehow the Gemfile,
execute:

```
gem update
```

1. Install or update the project
Anytime you change the _config.yml or any other configuration its recomended to:

* For starters to execute:

```
bundle install
```

* For the rest of the time, please execute:

```
bundle update
```

1. Run the project

You can run it straight forward with the command:

```
jekyll serve
```

But to avoid any possible conflict due to the missmatch of Ruby dependencies 
please excecute:

```
bundle exec jekyll serve
```
