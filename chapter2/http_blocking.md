# HTTP队头阻塞问题

通常我们提到队头阻塞，一般指的是TCP的队头阻塞，因为HTTP底层依赖的是TCP，所以HTTP/1.1包括HTTP/2也存在队头阻塞问题。