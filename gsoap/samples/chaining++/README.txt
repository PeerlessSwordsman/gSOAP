
How to combine multiple C++ services into one server executable?

The following method works well for all simple services, but not for
plugin-based behaviors as plugin states are not propagated when chaining. In
that case, please read suggestions further below to use a different option.

Run wsdl2h on each WSDL, with "wsdl2h name.wsdl" to produce name.h

Run "soapcpp2 -i -S -qName name.h" to produce a service class in a C++
namespace. The files generated by this command that you need are
NamexxxService.h, NamexxxService.cpp, and NameC.cpp

Run "soapcpp2 -CS -penv env.h" on an empty env.h (or an env.h with shared SOAP
Header and SOAP Fault definitions) to produce envC.cpp for Header and Fault
processing.

Compile and link all these files together with libgsoap++ (or stdsoap2.cpp).

Implement service chaining as follows:

Name1::xxxService s1;
Name2::yyyService s2;
Name3::zzzService s3;

s1.bind(NULL, 8080, 100);
s1.accept();
if (soap_begin_serve(&s1))
  ... error
else if (s1.dispatch() == SOAP_NO_METHOD)
{
  soap_copy_stream(&s2, &s1);
  if (s2.dispatch() == SOAP_NO_METHOD)
  {
    soap_copy_stream(&s3, &s2);
    if (s3.dispatch() == SOAP_NO_METHOD)
      ... error
    soap_free_stream(&s3);
  }
  soap_free_stream(&s2);
}
s1.destroy();
s2.desotry();
s3.destroy();

The above assumes soapcpp2 option -i is used, which means that each service
object has its own engine state. This does not work well with plugins as it is
tricky to ensure the plugin states are kept in sync.

You can use soapcpp2 with option -j instead of -i: run "soapcpp2 -j -S -qName
name.h" to produce a service class in a C++ namespace such that the instance
has a soap context that can be shared. This allows plugins to be registered
just once and shared across the services. The code is changed to:

struct soap *soap = soap_new();
Name1::xxxService s1(soap);
Name2::yyyService s2(soap);
Name3::zzzService s3(soap);

soap_bind(soap, NULL, 8080, 100);
soap_accept(soap);
if (soap_begin_serve(soap))
  ... error
else if (s1.dispatch() == SOAP_NO_METHOD)
{
  if (s2.dispatch() == SOAP_NO_METHOD)
  {
    if (s3.dispatch() == SOAP_NO_METHOD)
      ... error
  }
}
soap_destroy(soap);
soap_end(soap);
soap_free(soap); // only safe when s1, s2, s3 are also deleted
