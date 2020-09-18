---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: 2020-03-15T12:00:00
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- golang
- ginkgo
- filesystem
- afero
title: 'GO: How to mock filesystem code'
type: post
---

Go code which depends on the file system contents like NUMA nodes, CPUs,
interfaces associated with NUMA nodes can be tested with multiple ways.
One posibility is to use isolate all file open code separately and get the
contents and pass it to the actual functions which has the processing logics.

Alternatively, it also possible to mock files using a temproary filesystem
only for tesing using ```spf13/afero``` project. It provides a wraper for
genric ```io``` and ```ioutil``` packages. But in the GO test files, this
wrapper can be altered to a local Memory based filesystem and the required
files and content could be written on the tests itself.

Here is an example of the GO file which reads the cpulist from NUMA node.

```
package main

import (
	"flag"
	"fmt"
	"strings"

	iowrap "github.com/spf13/afero"
)

var (
	FS     iowrap.Fs
	FSUtil *iowrap.Afero
)

func init() {
	FS = iowrap.NewOsFs()
	FSUtil = &iowrap.Afero{Fs: FS}
}

func main() {
	Calc()
}

func Calc() (string, error) {
	content, err := FSUtil.ReadFile("/sys/devices/system/node/node1/cpulist")
	if err != nil {
		fmt.Println("ERROR:", err)
		return "", err
	}
	ret := strings.TrimSuffix(string(content), "\n")
	fmt.Println("Content:", ret)
	return ret, nil
}
```


The function ```Calc``` can be tested by mocking the filesystem to a local
memory based filesystem like ```NewMemMapFs```, which will allows us to create
files with specific contents as pwer the test needs. This will avoid the need
to restructure the code to separate the file read logics. Anf also any error
reading file could also be tested to see if valid error messages are provided.

Below snippet is the test file for the above mentioned source file:

```
package main_test

import (
	. "github.com/krsacme/go-play-ground"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	iowrap "github.com/spf13/afero"
)

func init() {
	FS = iowrap.NewMemMapFs()
	FSUtil = &iowrap.Afero{Fs: FS}

	FS.MkdirAll("/sys/devices/system/node/node1", 0755)
	iowrap.WriteFile(FS, "/sys/devices/system/node/node1/cpulist", []byte("0-7\n"), 0644)
}

var _ = Describe("Main", func() {

	Describe("calc", func() {
		Context("calc function", func() {
			It("should return numa nodes", func() {
				Expect(Calc()).To(Equal("0-7"))
			})
		})
	})

```
