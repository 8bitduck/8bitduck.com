+++
date = "2015-01-07T15:44:36-08:00"
draft = false
title = "Install gitlab-ci-runner on OSX Yosemite (10.10)"

+++

Installing the gitlab-ci-runner on OSX is no small feat, mainly because of the icu4c dependency and no setsid. Since this is a build box and not a development machine, I didn't want to install homebrew since it wasn't really designed around a multi-user environment. Also, don't forget that OSX comes with Ruby 2.0.0 already installed.

#### Install Xcode

We need the compilers from Xcode to build icu4c, setsid and other ruby gems.

#### Create a user

I called it gitlabci (if you choose something different, you will need to adjust the instructions below).

#### Install icu4c

    curl -L -O http://download.icu-project.org/files/icu4c/54.1/icu4c-54_1-src.tgz
    tar xfvz icu4c-54_1-src.tgz
    cd icu/source
    ./configure --disable-samples --disable-tests --enable-static
    make
    sudo make install

#### Install setsid

    https://github.com/jerrykuch/ersatz-setsid.git
    cd ersatz-setsid/
    clang -o setsid setsid.c
    sudo mv setsid /usr/local/bin/setsid

#### Install bundler

    sudo gem install bundler

#### Download and Install gitlab-ci-runner

    # login as the runner user
    sudo -u gitlabci -i

    cd ~/
    git clone https://gitlab.com/gitlab-org/gitlab-ci-runner.git
    cd gitlab-ci-runner
    bundle config build.charlock_holmes --with-icu-dir=/usr/local
    bundle install --deployment
    bundle exec ./bin/setup
    
    # logout of the gitlabci user
    exit

#### Create the Startup (launchd) Script

This "script" that will start the runner and keep it running. Launchd is what OSX uses instead of init.d scripts.

    cat > com.gitlab.ci.runner.plist <<-EOF
    <?xml version="1.0" encoding="UTF-8"?> 
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd"> 
    <plist version="1.0">
    <dict>
      <key>Label</key>
      <string>com.gitlab.ci.runner</string>
      
      <key>ProgramArguments</key>
      <array>
        <string>/usr/bin/bundle</string>
        <string>exec</string>
        <string>bin/runner</string>
      </array>
      
      <key>WorkingDirectory</key>
      <string>/Users/gitlabci/gitlab-ci-runner</string>

      <key>UserName</key>
      <string>gitlabci</string>

      <key>GroupName</key>
      <string>staff</string>
      
      <key>RunAtLoad</key>
      <true/>

      <key>KeepAlive</key>
      <true/>
    </dict>
    </plist>
    EOF
    
    sudo chown root:wheel com.gitlab.ci.runner.plist
    sudo mv com.gitlab.ci.runner.plist /Library/LaunchDaemons

This would be a lot simpler if the gitlab-ci-runner was written in go, it would be much easier to get a runner going and easier to support different OS platforms.
