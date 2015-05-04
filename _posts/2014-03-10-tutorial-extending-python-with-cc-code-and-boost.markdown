---
layout: post
title:  "Tutorial: Extending Python with C/C++ code and Boost"
date:   2014-03-10 04:35:0
---

At x20x we deal with a lot of APIs, and even more data. And while python is usually fast enough, there are parts that require blazing fast performance. This is where a programmer is left with couple options:<br />
– Optimize python routines<br />
– Rewrite parts of your code to Cython<br />
– Write the required code in C/C++ and expose this functionality to python.<br />
Generally speaking, optimizing python routines is the most important step but it will also get you only to a certain point after which you cannot improve it anymore. When that is not enough you have to either write the parts in Cython or write them in C/C++ and then include them inside your Python. I personally dislike Cython (reasons won’t be included in this article); so I strongly lean toward C++ extensions. That also comes with multiple options, but in this article I will explain the easiest, and my favorite, way to do it – with Boost.Python.

<!--more-->

We should start by writing the crucial part of the code in C++ and make sure that it works. For this tutorial, we will use API client for [http://friendlyproxies.com/][proxies]. We use them at X20X to scrap google SERPS (they are awesome, make sure to check them out if you are in need of heavy googling without hassles of getting banned and rotating long proxy list) and you can find the code in this handy gist (it isn’t the production code we use, but it’s much more readable than plain C mess we use): [https://gist.github.com/Puciek/9465889#file-client-cpp][gist]. Download it and fire it up in your favorite editor. And with that, let’s go over the code.
First we define a function to handle compression.

{% highlight c++ %}
std::string decompress_string(const std::string& str)
{
    z_stream zs;                        // z_stream is zlib's control structure
    memset(&zs, 0, sizeof(zs));
 
    if (inflateInit(&zs) != Z_OK)
        throw(std::runtime_error("inflateInit failed while decompressing."));
 
    zs.next_in = (Bytef*)str.data();
    zs.avail_in = str.size();
 
    int ret;
    char outbuffer[32768];
    std::string outstring;
 
    // get the decompressed bytes blockwise using repeated calls to inflate
    do {
        zs.next_out = reinterpret_cast<Bytef*>(outbuffer);
        zs.avail_out = sizeof(outbuffer);
 
        ret = inflate(&zs, 0);
 
        if (outstring.size() < zs.total_out) {
            outstring.append(outbuffer,
                zs.total_out - outstring.size());
        }
 
    } while (ret == Z_OK);
 
    inflateEnd(&zs);
 
    if (ret != Z_STREAM_END) {          // an error occurred that was not EOF
        std::ostringstream oss;
        oss << "Exception during zlib decompression: (" << ret << ") "
            << zs.msg;
        throw(std::runtime_error(oss.str()));
    }
 
    return outstring;
}
{% endhighlight %}
Which takes a ZLIB compressed std::string as input and returns uncompressed one by leveraging Zlib.h.

Next we have our function of interest:

{% highlight c++ %}
std::string query_google(string url, string referer, string region, string useragent, string priority, string key)
{
    Json::Value details;
    details["url"] = url;
    details["referer"] = referer;
    details["region"] = region;
    details["useragent"] = useragent;
    details["priority"] = priority;
    Json::Value request;
    request["key"] = key;
    request["request"] = details;
 
    const char* host = "api.proxyrotation.com";
    const char* port = "9005";
 
    using namespace boost::asio;
    io_service io_service;
    ip::tcp::resolver resolver(io_service);
    ip::tcp::resolver::query query(host, port);
    ip::tcp::resolver::iterator endpoint_iterator = resolver.resolve(query);
    ip::tcp::resolver::iterator end;
 
    ip::tcp::socket socket(io_service);
    boost::system::error_code error = boost::asio::error::host_not_found;
    socket.connect(*endpoint_iterator++, error);
    Json::StyledWriter writer;
    socket.write_some(boost::asio::buffer(writer.write(request)));
 
    boost::array<char, 120000> buf;
    std::string response;
    size_t len = 0;
    while (len = socket.read_some(boost::asio::buffer(buf), error)) {
        std::string sbuf(buf.begin(), len);
        response.append(sbuf);
    }
    socket.close();
    return decompress_string(response);
}
{% endhighlight %}

Which takes a lot of arguments, builds a json request based on them, sends it to the API endpoint, reads the data back from socket and returns uncompressed result. It’s pretty basic example of how to build a TCP client with Boost, nothing special in here.

Now we have all the logic we need to operate with API, all is left is to expose it to python. Luckily enough, it’s very easy to do with Boost!
All you have to do is to include boost/python.hpp into your application and then it’s a matter of very simple and self-explanatory piece of code:

{% highlight c++ %}
BOOST_PYTHON_MODULE(client)
{
using namespace boost::python;
def("query_google", query_google);
}
{% endhighlight %}
And if we wanted to expose a class, it is almost as easy:


{% highlight c++ %}
BOOST_PYTHON_MODULE(foo)
{
class_&lt;fooClass&gt;("fooClass")
.def("bar", &amp;fooClass::bar)
.def("qux", &amp;fooClass::qux)
;
}
{% endhighlight %}
Now, with the code ready, we have to compile and link it, so let’s get to it. Make sure that you got jsoncpp, boost_system and boost_python installed; otherwise, you will not have a good time.

1
{% highlight bash %}
g++ -c -fPIC client.cpp -o client.o -I /usr/include/python2.7/
{% endhighlight %}
The -I argument and following path is not required if your g++ is running on more liberal settings. I prefer to have longer compile commands, and thus impose tighter control over what is going in. If you want to build the code against another version of python, just change the path accordingly.
Assuming that no error shows up (if there are some, try upgrading GCC to 4.6/4.7 branch), we have successfully compiled our client and all left for there is to link it all together.

{% highlight bash %}
g++ -shared -Wl,-soname,client.so -o client.so client.o -lpython2.7 -lboost_python -lboost_system -ljsoncpp
{% endhighlight %}
Make sure that boost_python and the same version which you used for compilation is included, otherwise you will run into missing symbols or segfaults.

Now with code compiled and linked into a handy .so, all there is left is to test it out:

{% highlight python %}
azureuser@MAI-R3-DED24:~/cppembed# python
Python 2.7.3 (default, Feb 27 2014, 19:58:35)
[GCC 4.6.3] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import client
>>> len(client.query_google("https://www.google.co.uk/search?q=magic", "http://google.com", "uk", "Mozilla/5.0 
(Windows NT 5.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/31.0.1650.16 Safari/537.36)", "1", 
"youwillneedakeytoactullygetthedata"))
270422
{% endhighlight %}

That looks like a success, we’ve queries the friendlyproxies API and received a serp which weights 270422 bytes. Sounds about right!

And that concludes this very basic introduction to exposing C++ functionality into Python. There is much more which Boost offers in that department and you can read about it in the documentation. And as always, remember to subscribe for more of the good, and leave your questions, flames, ideas and thoughts in the comment section below.

[proxies]: http://friendlyproxies.com/
[gist]: https://gist.github.com/Puciek/9465889#file-client-cpp