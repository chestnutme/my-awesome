[Understanding nil - GopherCon2016](https://www.youtube.com/watch?v=ynoY2xz-F8s)

## what is nil
1. etymology
    nil : latin `nihil` - nothing
    null : latin `ne + ullus` - not any
    none : eng `en + an` - not one
2. history
    C.A.R Hoare <Communicating Sequential Process> 1965
    nil in go : zero value

3. zero value in golang
    1. value type : bool -> false, numbers -> 0, string -> ""
    2. reference type : slice, map, channel-> nil
    3. other type : pointer, funciton, inteface -> nil
    4. struct type : zero value of its each field

4. type of nil
>> "unless the value is the predeclared identifier nil, which has no type" - go spec

**nil has no type, nil is predeclared identifier, not a key word**
which means you can define it
```golang
val nil = error.New("bingo")
```

5. nil for each type in golang
    1. pointer : address of a memory address, but no arithmetic -> memory safety and for gc; nil pointer -> point to nothing
    2. slice : [*elem | len | cap] ; nil slice -> [nil, 0, 0], no backing array
    3. map, chan, func -> ptr; nil it -> nil pointer
    4. interface: [type, value] ; nil interface -> [nil, nil] = nil ; interface [*type1, nil] != nil 
    ```golang
    // error is interface in golang
    // do not declare concrete error vars(also in return value)
    // should declare error interface var, if no error, return nil
    func do() error {
        var err *doError
        return err
    }

    func main() {
        err := do()              // err[*doError, nil]
        fmt.Printfln(err == nil) // no equal!
    }
    ```

    ```golang
    func do() *doError {  // *doError is nil
        return nil;
    }
    
    func wrapDo() error { // error (*doError, nil)
        return do();
    }

    func main() {
        err := wrapDo()
        fmt.Println(err == nil) // no equal!
    }
    ```

6. nil is useful
```golang
var p *int
p == nil    // true
*p          // panic: nil pointer reference


type tree struct {
    val int
    l *tree
    r *tree
}

func (t *tree) Sum() int {
    if(t == nil) {
        return 0;
    }

    sum := t.val
    if(t.l != nil) {
        sum += t.l.Sum()
    }

    if(t.r != nil) {
        sum += t.r.Sum()
    }

    return sum
}

// nil pointer can still call method
func (t *tree) Sum() int {
    if(t == nil) {
        return 0
    }

    return t.v + t.l.Sum() + t.r.Sum()
}


var s []byte
len(s)          // 0
cap(s)          // 0
for range s     // iterate zero times
s[1]            // panic: index out of range

append(s, `c`)  // ok

var m [string]string
len(m)          // 0
for range m     // iterate zero times
v, ok := m["no key"]    // "", false
m["key"] = "value"      // panic: assign to entry in nil map

func NewGet(url string, headers map[string]string) (*http.Request, error) {
    req, err := http.NewRequest(http.MethodGet, url, nil)
    if err != nil {
        return nil, err
    }

    for k, v := range headers {
        req.Header.Set(k, v);
    }

    return req, nil
}

// nil map is a read-only empty map
req, err := NewGet("http://google.com", nil)

var c chan int
<- c            // blocks forever
c <- x          // blocks forever
close(c)        // panic: close of nil channel

// set nil chan to disable select, instead of comsuming cpu when close(chan)
// see nil_channel.go

// nil function as default valus
func NewLogger(logger func(string, ...interface{})) {
    if logger == nil {
        logger = log.Printf
    }

    logger("init %s\n", os.Getenv("hostname"));
}

// nil interface[(type, value)] most used signal in GO
if err != nil {

}

// use as default value
// nil interface handler
http.HandleFunc("localhost:8888", nil)
```
