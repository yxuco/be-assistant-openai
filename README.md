# be-assistant-openai

This project contains a notebook [BEPOC](./BEPOC.ipynb) that uses an [OpenAI](https://platform.openai.com/apps) assistant to write rules and rule-functions for [TIBCO BusinessEvents (BE)](https://docs.tibco.com/products/tibco-businessevents-enterprise-edition).

The [notebook]((./BEPOC.ipynb)) contains code for creating and using the OpenAI assistant, as well as sample requests and responses for writing BE rules and rule-functions.  You may read the samples in the file to evaluate the quality of AI-generated code.

It appears that the future of software engineering will be done by AI prompts written in natural language, instead of computer code in a specialized programming language.

## Setup OpenAI API Account

To see how the code in the notebook works, you will need to sign up an API account in [OpenAI](https://platform.openai.com/apps).  Make sure that you use the `API` account, not `ChatGPT` account.  With an API account, you pay as you go, and it is usually cheaper if you do not use a lot of data.  The samples in this notebook will not cost more than `$2.00`. 

After you login to OpenAI, you can create a test project, and then create a new [API Key](https://platform.openai.com/api-keys), which is required to setup your development environment.

## Setup Python Development Environment

I use [jupyter](https://jupyter.org/install) notebook for Python development, and use [pyenv](https://github.com/pyenv/pyenv) to manage installed versions of Python.

On macOS, you may setup your development environment using the following steps.

```
# install pyenv on macOS
brew update
brew install pyenv

# or upgrade pyenv if it is already installed
# brew upgrade pyenv

# find and install latest python version
pyenv versions
pyenv install -l
pyenv install 3.12.4

# edit ~/.pyenv/version to set python version 3.12

# install openAI
pip install openai

# install notebook
pip install notebook

# set OpenAI key and start notebook
export OPENAI_API_KEY="sk-proj-...."
jupyter notebook
```

## Artifacts

Documentations of BE library functions are collected in the following JSON documents, which will be used as context for AI models.

* [Standard.json](Standard.json)
* [RDBMS.json](./RDBMS.json)
* [CEP.json](./CEP.json)

If fine-tuning is required, the file [tuning-samples.jsonl](./tuning-samples.jsonl) shows the format of fine-tuning data for OpenAI models.

Following docs describe how code assistant can be used to generate rules and functions for TIBCO BusinessEvents.

* [openaiAPI.md](./openaiAPI.md) describes how the Python API of OpenAI is used to generate BE artifacts as implemented in [BEPOC.ipynb](./BEPOC.ipynb).
* [vsCodeExtension.md](./vsCodeExtension.md) describes how VS Code can use an AI model running either in cloud or locally to generate BE artifacts.