## I/O 프로젝트 개선하기

반복자에 대한 새로운 지식을 사용하여 12장의 I/O 프로젝트의 코드들을 더 깔끔하고
간결하게 개선할 수 있습니다. 반복자를 사용하여 어떻게 `Config::new` 함수와
`search` 함수의 구현을 개선할 수 있는지 살펴봅시다.


### 반복자를 사용하여 `clone` 제거하기

리스트 12-6 에서, `String` 값의 슬라이스를 받고 슬라이스를 인덱싱하고 복사
함으로써 `Config` 구조체의 인스턴스를 생성하였고, `Config` 구조체가 이 값들을
소유하도록 했습니다. 리스트 13-24 에서는 리스트 12-23 에 있던것 처럼
`Config::new` 함수의 구현을 다시 재현 했습니다:

<span class="filename">파일명: src/lib.rs</span>

```rust,ignore
impl Config {
    pub fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config { query, filename, case_sensitive })
    }
}
```

<span class="caption">리스트 13-24: 리스트 12-23 의 `Config::new` 함수 재현
</span>

그 당시, 비효율적인 `clone` 호출에 대해 걱정하지 말라고 얘기 했으며 미래에
없앨 것이라고 했습니다. 자, 그때가 되었습니다!

`String` 요소들의 슬라이스를 `args` 파라미터로 받았지만 `new` 함수는 `args` 를
소유하지 않기 때문에 `clone` 이 필요했습니다. `Config` 인스턴스의 소유권을
반환하기 위해, `Config` 의 `query` 와 `filename` 필드로 값을 복제 함으로써
`Config` 인스턴스는 그 값들을 소유할 수 있습니다.

반복자에 대한 새로운 지식으로, 인자로써 슬라이스를 빌리는 대신 반복자의 소유권을
갖도록 `new` 함수를 변경할 수 있습니다. 슬라이스의 길이를 체크하고 특정 위치로
인덱싱을 하는 코드 대신 반복자의 기능을 사용할 것 입니다. 이것은 반복자가 값에
접근 할 것이기 때문에 `Config :: new` 함수가 무엇을 하는지를 명확하게 해줄
것입니다.

`Config::new` 가 반복자의 소유권을 갖고 빌린 값에 대한 인뎅싱을 사용하지 않게
된다면, `clone` 을 호출하고 새로운 할당을 만드는 대신 `String` 값들을 반복자에서
`Config` 로 이동할 수 있습니다.


#### 반환된 반복자를 바로 사용하기

Open your I/O project’s *src/main.rs* file, which should look like this:

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
}
```

We’ll change the start of the `main` function that we had in Listing 12-24 at
to the code in Listing 13-25. This won’t compile until we update `Config::new`
as well.

<span class="filename">Filename: src/main.rs</span>

```rust,ignore
fn main() {
    let config = Config::new(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
}
```

<span class="caption">Listing 13-25: Passing the return value of `env::args` to
`Config::new`</span>

The `env::args` function returns an iterator! Rather than collecting the
iterator values into a vector and then passing a slice to `Config::new`, now
we’re passing ownership of the iterator returned from `env::args` to
`Config::new` directly.

Next, we need to update the definition of `Config::new`. In your I/O project’s
*src/lib.rs* file, let’s change the signature of `Config::new` to look like
Listing 13-26. This still won’t compile because we need to update the function
body.

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore
impl Config {
    pub fn new(mut args: std::env::Args) -> Result<Config, &'static str> {
        // --snip--
```

<span class="caption">Listing 13-26: Updating the signature of `Config::new` to
expect an iterator</span>

The standard library documentation for the `env::args` function shows that the
type of the iterator it returns is `std::env::Args`. We’ve updated the
signature of the `Config::new` function so the parameter `args` has the type
`std::env::Args` instead of `&[String]`. Because we’re taking ownership of
`args` and we’ll be mutating `args` by iterating over it, we can add the `mut`
keyword into the specification of the `args` parameter to make it mutable.

#### Using `Iterator` Trait Methods Instead of Indexing

Next, we’ll fix the body of `Config::new`. The standard library documentation
also mentions that `std::env::Args` implements the `Iterator` trait, so we know
we can call the `next` method on it! Listing 13-27 updates the code from
Listing 12-23 to use the `next` method:

<span class="filename">Filename: src/lib.rs</span>

```rust
# fn main() {}
# use std::env;
#
# struct Config {
#     query: String,
#     filename: String,
#     case_sensitive: bool,
# }
#
impl Config {
    pub fn new(mut args: std::env::Args) -> Result<Config, &'static str> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config { query, filename, case_sensitive })
    }
}
```

<span class="caption">Listing 13-27: Changing the body of `Config::new` to use
iterator methods</span>

Remember that the first value in the return value of `env::args` is the name of
the program. We want to ignore that and get to the next value, so first we call
`next` and do nothing with the return value. Second, we call `next` to get the
value we want to put in the `query` field of `Config`. If `next` returns a
`Some`, we use a `match` to extract the value. If it returns `None`, it means
not enough arguments were given and we return early with an `Err` value. We do
the same thing for the `filename` value.

### Making Code Clearer with Iterator Adaptors

We can also take advantage of iterators in the `search` function in our I/O
project, which is reproduced here in Listing 13-28 as it was in Listing 12-19:

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }

    results
}
```

<span class="caption">Listing 13-28: The implementation of the `search`
function from Listing 12-19</span>

We can write this code in a more concise way using iterator adaptor methods.
Doing so also lets us avoid having a mutable intermediate `results` vector. The
functional programming style prefers to minimize the amount of mutable state to
make code clearer. Removing the mutable state might enable a future enhancement
to make searching happen in parallel, because we wouldn’t have to manage
concurrent access to the `results` vector. Listing 13-29 shows this change:

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents.lines()
        .filter(|line| line.contains(query))
        .collect()
}
```

<span class="caption">Listing 13-29: Using iterator adaptor methods in the
implementation of the `search` function</span>

Recall that the purpose of the `search` function is to return all lines in
`contents` that contain the `query`. Similar to the `filter` example in Listing
13-19, this code uses the `filter` adaptor to keep only the lines that
`line.contains(query)` returns `true` for. We then collect the matching lines
into another vector with `collect`. Much simpler! Feel free to make the same
change to use iterator methods in the `search_case_insensitive` function as
well.

The next logical question is which style you should choose in your own code and
why: the original implementation in Listing 13-28 or the version using
iterators in Listing 13-29. Most Rust programmers prefer to use the iterator
style. It’s a bit tougher to get the hang of at first, but once you get a feel
for the various iterator adaptors and what they do, iterators can be easier to
understand. Instead of fiddling with the various bits of looping and building
new vectors, the code focuses on the high-level objective of the loop. This
abstracts away some of the commonplace code so it’s easier to see the concepts
that are unique to this code, such as the filtering condition each element in
the iterator must pass.

But are the two implementations truly equivalent? The intuitive assumption
might be that the more low-level loop will be faster. Let’s talk about
performance.
