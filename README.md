# PostgreSQL documentation

[![License](https://img.shields.io/github/license/mnementh64/postgresql-doc.svg)](LICENSE)

This technical doc has been built by a non DBA PostgreSQL. I have used PostgreSQL a lot and I wanted to learn more about it.

Here is the result of many blogs I read, many confs I have seen, many tests I have done, ...

[Enjoy on Github !](https://mnementh64.github.io/postgresql-doc/)

## To see the doc

It's a [MkDocs](http://www.mkdocs.org/) documentation based on Markdown format.

To display it, go to this directory.

I you have installed MkDocs, run :

	mkdocs serve

Or use :

	docker run -d --rm -p 8000:8000 -e UID=$(id -u) -e GID=$(id -g) -v $PWD:/mkdocs mnementh64/docker-mkdocs-serve

And, in your favorite browser, go to : `http:localhost:8000`.
