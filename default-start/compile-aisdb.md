---
hidden: true
cover: >-
  https://images.unsplash.com/photo-1515879218367-8466d910aaa4?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHw4fHxwcm9ncmFtbWluZ3xlbnwwfHx8fDE3MjMzMDE5OTV8MA&ixlib=rb-4.0.3&q=85
coverY: 0
---

# ⚙️ Compile AISdb

### #️⃣Source Code Installation

This tutorial describes the source code setup in Pycharm for Windows, Linux, and Mac.

### Windows <a href="#windows" id="windows"></a>

1. **Installing Python** Download and install any python >= 3.8 [https://www.python.org/downloads/](https://www.python.org/downloads/) ensure that path is added to environmental variables. \[Python 3.12.0]
2. **Installing Pycharm** Download and install Pycharm which supports VCS
3. **Installing Rust** Many basic functions are implemented in Rust for fast execution. The Rust code is called from Python using Maturin. We will discuss about Maturin in the later section of this tutorial but first let's start downloading and installing Rust: [https://www.rust-lang.org/tools/install](https://www.rust-lang.org/tools/install) Run the .exe and Choose the option **(1)** It will install Rust as well as Visual Studio C++ tools. After completing, run the command prompt and run this command to check if it is successfully installed \[rustc 1.73.0]
4. **Installing StrawberryPerl** The project uses the OpenSSL library and for that, Perl is needed. Don't worry you don't need to buy strawberries to install it just click the link [https://strawberryperl.com](https://strawberryperl.com/). Download and install \[Perl v5.38.0]
5. **Restart the computer** Sometimes it is better to restart the computer so that installed compilers and tools are perfectly in sync with the operating system.

_**Generating Access Token from Github**_ Generating a token is required when accessing any repository from Github _Account_ ▶ _Setting_ ▶ _Developer Settings_ ▶ _Personal access token_ ▶

**Open Project in Pycharm**

select the _Get from VCS_

1.
2. Paste the URL then it will ask for an access token or an account link. Paste the access token that was generated in the previous step.

**Creating a Python Virtual Environment**

It is a standard practice to work on a virtual environment for each individual project so that dependencies don't conflict.

Open Terminal in Pycharm

check the python first by executing the command

creating a virtual environment

{% code lineNumbers="true" %}
```
python -m venv env_aisdb_client 
```
{% endcode %}

activating the environment

{% code lineNumbers="true" %}
```
.\env_aisdb_client\Scripts\activate
```
{% endcode %}

When you create a virtual environment, you will see a folder has been created in the project with name `env_aisdb_client.` It contains our python environment.

After activating the environment, a Python library needs to be installed

Now, almost everything is done with the setup. We need to compile code to proceed and for that run this command

{% code lineNumbers="true" %}
```
maturin develop --release --extras=test,docs
```
{% endcode %}

or

make sure your current directory in the cmd should be the directory of the project

Tada! Everything is completed now.

If the develop is getting error about unavailability of unzip command then download \[[https://sourceforge.net/projects/gnuwin32/](https://sourceforge.net/projects/gnuwin32/)] unzip and set the bin path into environmental variables

***

### Linux <a href="#linux" id="linux"></a>

{% code lineNumbers="true" %}
```
# You first need to install the rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Specify Rust version and check
rustup install nightly-2023-07-15 && \
rustup default nightly-2023-07-15 && \
rustc --version

# Create your VEnv
python -m venv AISdb
source ./AISdb/bin/activate

# You will require Maturin to build the package
pip install maturin
pip install patchelf

# Clone the source code and navigate to the package root
git clone https://github.com/AISViz/aisdb-client.git && cd aisdb-client

# Build and install AISdb to your VEnv
maturin develop --release --extras=test,docs
```
{% endcode %}

***

### Mac <a href="#mac" id="mac"></a>

[PreviousDownloading Metadata](/broken/pages/ssgzutCN5OoYJPu2OFkf)[NextIntegrating with ML/DL](/broken/pages/NgZuEhiumqltg3SyRuxK)

Last updated 6 months ago
