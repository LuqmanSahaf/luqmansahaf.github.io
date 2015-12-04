---
layout: post
title: Iteration Order of Maps in Go and Test Cases
excerpt_separator: <!--excerpt_end-->
---

_This is a riddle about how I came about a problem while writing test cases in Go for a function that involved map data structure._

<!--excerpt_end-->

I faced this problem when I started writing test cases for a project written in Go. The problem appeared when I wrote a test case for a routine which took a map as input and return a string based on the content of the map.

The problem arised due to the iteration order of maps in Go. The order of iteration of maps is _sort of_ [_random_](https://news.ycombinator.com/item?id=7655948). 

To illustrate the problem, take a look at the following routine:

```go
/* This is a simple illustration. */
func doSomething(in map[string]string) (res string) {
  for key, value := range in {
    res += key + "=" + value + " "
  }
  return
}
```

Now, if iteration order was the insertion order, then the test case would simply be:

```go
func TestDoSomething(t *testing.T) {
  cases := []struct{
    in map[string]string
    wants string
  }{
    {
      map[string]string{},
      "",
    },
    {
      map[string]string{"a":"1", "b":"2", "c":"3", "d":"4"},
      "a=1 b=2 c=3 d=4 ",
    },
  }

  for _,c := range cases {
    got := doSomething(c.in)
    if got != c.wants {
      t.Errorf("doSomething(c.in) == %q \n wanted: %q", got, c.wants)
    }
  }
}
```

Running this test case resulted in success some times and failed otherwise because of randomness in iteration.
The solution to this problem is to try all combinations for the input. In the above case, it will yield 24 (4!, assuming full uniformly random iteration which is not the [case](https://news.ycombinator.com/item?id=7655948),) possible test results (`wants`). Enter _induction_. After much thought, I realised that it was simply a case of induction. I just have to test for _n_, and _n+1_ to prove that the routine is working fine. That is, reduce the input size and try for all the possibilites, which are quite small in number as compared to above cases. **Note** that it is only true, if it does not matter that in what order do the entries of map appear in output.

```go
func TestDoSomething(t *testing.T) {
  cases := []struct{
    in map[string]string
    wants []string
  }{
    {
      map[string]string{},
      []string{""},
    },
    {
      map[string]string{"a":"1"},
      []string{"a=1 "},
    },
    {
      map[string]string{"a":"1", "b":"2"},
      []string{
        "a=1 b=2 ",
        "b=2 a=1 ",
      },
    },
  }

  for _,c := range cases {
    got := doSomething(c.in)
    pass := false
    for _, w := range c.wants {
      if w == got {
        pass = true
        break
      }
    }
    if !pass {
      t.Errorf("doSomething(c.in):\n --> got: %q\n --> wanted any of: %q", got, c.wants)
    }
  }
}
```

_Happy testing!_