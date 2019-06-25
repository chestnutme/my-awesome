[Understanding nil - GopherCon2016](https://www.youtube.com/watch?v=ynoY2xz-F8s)
1. zero value for pointer and reference type(slice/map/channel), function, interface
2. nil is useful
   * nil pointer as method receiver
   * nil slice is valid zero value
   * nil map is perfect read-only zero value
   * essential for some concurrency patterns[close(chan) is not enough when select, need set nil]
   * nil functions as default value and for completenesss
   * nil interface[(type, value)] most used signal in GO(err != nil, nil != (*Person)nil)
