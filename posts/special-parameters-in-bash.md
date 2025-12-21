---
title: Special $@ and $* Parameters in bash
published: true
description: Difference between Special $@ and $* Parameters in bash.
tags: 'bash,gnulinux,linux'
cover_image: 'https://raw.githubusercontent.com/cod3mas0n/Documents/refs/heads/main/posts/assets/special-parameters-in-bash.webp'
canonical_url: null
id: 2319639
date: '2025-03-08T22:48:26Z'
---

There is a say no difference between `$@` and `$*` in Shell! What ?!!

Have you ever thought if there is no difference, why there are two of them? it could be one `$@` or `$*` , Do You Agree?

So Let’s see What the difference is . Create a script and try it one by one

First One `$@` :

```bash
#! /bin/bash

MAIN()
{
        echo "First Parameter is $1"
}

MAIN $@
```

Execute it like below and give the script some arguments:

```bash
$ ./Diff.sh "Jhon Smith" "Marry Ann"
First Parameter is Jhon

```

Second One `$*` :

```bash
#! /bin/bash

MAIN()
{
        echo "First Parameter is $1"
}

MAIN $*
```

The result will be :

```bash
$ ./Diff.sh "Jhon Smith" "Marry Ann"
First Parameter is Jhon
```

Far Now There is no difference, maybe those people were right!

Let us go further and try them with double quotes `"$@"` :

```bash
#! /bin/bash

MAIN()
{
        echo "First Parameter is $1"
}

MAIN "$@"
```

```bash
$ ./Diff.sh "Jhon Smith" "Marry Ann"
First Parameter is Jhon Smith
```

The second one in double quotes `"$*"` :

```bash
#! /bin/bash

MAIN()
{
        echo "First Parameter is $1"
}

MAIN "$*"
```

```bash
$ ./Diff.sh "Jhon Smith" "Marry Ann"
First Parameter is Jhon Smith Marry Ann
```

So be careful when you use double quotes and tell those people there is a difference, explain to them shell is sensitive to double quotes.

## Resources

- [Gnu.org](https://www.gnu.org/software/bash/)
- [Bash Documents](https://www.gnu.org/software/bash/manual/)
- [Book](https://www.amazon.ca/Learning-bash-Shell-Unix-Programming/dp/0596009658)
- [Special Parameters](https://www.gnu.org/software/bash/manual/html_node/Special-Parameters.html)
