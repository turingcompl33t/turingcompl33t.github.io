
- Visit the [AWS management console](https://console.aws.amazon.com/?nc2=h_m_mc). Create an account if you do not already have one. Account creation requires a valid credit card number, but the resources we use for this class will all use the Free Tier so you won't be charged.
- Sign into the AWS management console.


- Click on the instance in the list of instances. Once the instance state transitions to `Running` you will be able to select _Connect_ from the options above the list of instances. On the next page, select _SSH Client_ and copy the command that appears beneath _Example_. You may have to change the argument to `-i` (the private key of the key pair specified during instance creation) if the name on your local system is different from the one shown, or if you need to add the path to the key file. If you use the default SSH key associated with your profile (e.g. _~/.ssh/id\_rsa_ if you use `ssh-keygen` on a UNIX-based system) you can skip the `-i` argument altogether (the default key will be used automatically).

```bash
# Update dependencies
$ sudo apt update

# Setup your private copy of BusTub
$ git clone --bare https://github.com/cmu-db/bustub.git bustub
$ cd bustub
$ git push --mirror <YOUR_PRIVATE_REPO>
$ cd .. && rm -rf bustub
$ git clone <YOUR_PRIVATE_REPO> bustub
$ cd bustub

# Install dependencies
$ sudo ./build_support/packages.sh

# Configure BusTub
$ mkdir build && cd build
$ cmake ..

# Check that everything works
$ make check-format
$ make check-lint
$ make check-clang-tidy
$ make check-tests
```

The final step is setup an editor or IDE to work with this remote environment. If you use a terminal-based editor like emacs or vim, no additional setup is required. If your prefer a graphical editor like Visual Studio Code or an IDE like CLion, it should be possible to configure these for remote development on this EC2 instance.
