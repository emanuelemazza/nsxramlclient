#nsxramlclient

This Python based client for NSX for vSphere 6.x gets its API structure information 
(e.g. URLs, parameters, schema, etc.) from a RAML file which describes the NSX for vSphere REST API. It has been developed and tested for use with VMware NSX for vSphere 6.x.

The latest version of the NSX for vSphere 6.x RAML file can be found at http://github.com/vmware/nsxraml

# How to install nsxramlclient

The following install instructions  are based on Ubuntu 14.04 LTS, but installations on other Linux distributions
or on the Apple MacOS should be relatively similar and should be familiar to those familiar with Python.

Check whether pip is installed
```sh
pip --version
```
If pip is not installed, install it with apt-get or the package manager appropriate for your operating system.
```sh
sudo apt-get update
sudo apt-get -y install python-pip
```
Now you can use pip to install the nsx raml client
```sh
sudo pip install nsxramlclient
```
In some cases the installation may fail because of missing dependencies. You may see the following message and will have to install the required packages 
```
ERROR: /bin/sh: 1: xslt-config: not found
** make sure the development packages of libxml2 and libxslt are installed **
```
This example shows installing the dependencies using the apt package manager and the apt-get command. Once dependencies are installed you can retry the pip installation of the nsxramlclient shown above.
```sh
sudo apt-get install libxml2-dev libxslt-dev python-dev zlib1g-dev
```

# Examples on how to use nsxramlclient

### Create a session object

It is required to create a session object with which you will interact with the NSX REST API. This session object will then expose the create, read, update and delete (CRUD) methods of each NSX object as well as some helper methods that will be useful.

```python
from nsxramlclient.client import NsxClient

nsxraml_file = '/raml/nsxraml/nsxvapiv614.raml'
nsxmanager = 'nsxmanager.invalid.org'
nsx_username = 'admin'
nsx_password = 'vmware'

client_session = NsxClient(nsxraml_file, nsxmanager, nsx_username, 
                           nsx_password, debug=False)
```
The NsxClient class has the following initialization parameters:
```python
"""
:param raml_file: 
This mandatory parameter is the RAML file used as the basis of all URL 
compositions. It allows the client to extract the body schema and convert the schema into python dictionaries.

:param nsxmanager: 
This mandatory parameter is either the hostname or IP Address of the NSX Manager appliance.

:param nsx_username: 
This mandatory parameter is the username on NSX Manager used for authentication to the NSX REST API running on the NSX Manager.

:param nsx_password: 
This mandatory parameter is the password of the user used for authentication to the NSX REST API running on the NSX Manager.

:param debug: Optional: 
If set to True, the client will print extensive HTTP session information to stdout. 
Default: False

:param verify: Optional: 
If set to True, the client will strictly verify the certificate passed by NSX Manager. It is recommmended in all production environments to use signed certificates for the NSX REST API. Please refer to the NSX for vSphere documentation for information on how to convert from the self-signed certificate to a signed certificate.
Default: False

:param suppress_warnings: Optional: 
If set to True, the client will print out a warning if NSX Manager uses a self signed certificate. 
Default: True

:return: Returns a NsxClient Session Object
"""
```
After you initialized a session object you have access to the following methods:

- create: 
Sends a HTTP POST to NSX Manager. More details will follow later in this readme.

- read: 
Sends a HTTP GET to NSX Manager

- update: 
Sends a HTTP PUT to NSX Manager

- delete: 
Sends a HTTP DELETE to NSX Manager

- view_response: 
Each of the above methods returns a Python OrderedDictionary with the HTTP Status code,
location header, NSX Object Id, eTag Header and Body. This method outputs the OrderedDict in human readable text to stdout.

- extract_resource_body_schema: 
This method will retrieve the body schema from the RAML File (if the
method has a body schema like most create methods), and will return a
template python dictionary that can be used to construct subsequent API calls.

- view_resource_body_schema: 
This method retrieves the body schema from the RAML file and outputs
it to stdout as a pretty printed XML document.

- view_body_dict: 
This method takes a body dictionary (any python dictionary), and outputs it in a human readable format to stdout.

- view_resource_display_names: 
This method outputs displayNames and descriptions of all resources in the RAML File with their associated URI & query parameters, additional headers, and what methods are supported.


### Use of the create, read, update and delete methods

```python
In [1]: client_session.read('vCenterStatus')
Out[2]: OrderedDict([('status', 200), ('body', {'vcConfigStatus': {'connected': 'true', 'lastInventorySyncTime': '1440444721014'}}), ('location', None), ('objectId', None), ('Etag', None)])
```
The create, read, update and delete methods return a Python OrderedDict with the following key/value pairs:
- status: The HTTP status code returned as an integer.
- body: The response body returned as a dict. If no body was returned the response will be ```None```
- location: If a location header is returned, this value will be the location URL as a string otherwise it will return ```None```
- objectId: If a location header is returned, the value of objectId will be the last part of the location url as a string otherwise it will return ```None```
- Etag: If a Etag header is returned, the value of Etag will be the content of the Etag header returned otherwise it will return ```None```

To output the response in a human readable format when working in an interactive session use the
```view_response``` method:
```python
In [3]: response = client_session.read('vCenterStatus')
In [4]: client_session.view_response(response)
HTTP status code:
200

HTTP Body Content:
{'vcConfigStatus': {'connected': 'true',
                    'lastInventorySyncTime': '1440445281484'}}

```
If a method needs a URI parameter to work, the NSX RAML Client will compose the URL based on the base URL, parent and child method URL and the supplied URI parameter. To supply a URI parameter, add a URI parameter dict to 
the call. You can supply multiple URI parameters in the call if needed.
```python
In [5]: response = client_session.read('vdnSegmentPool', 
                                       uri_parameters={'segmentPoolId': '2'})
In [6]: client_session.view_response(response)
HTTP status code:
200

HTTP Body Content:
{'segmentRange': {'begin': '5000',
                  'end': '10000',
                  'id': '2',
                  'name': 'legacy'}}

```
If a method supports one or more query parameters, you can supply those optional query parameters in your request, 
and the NSX RAML Client will add the query parameter for you. To use this pass a query parameter dict to the call:
```python
In [7]: response = client_session.read('nwfabricStatus', 
                                       query_parameters_dict={'resource': 
                                                              'domain-c1632'})
In [8]: client_session.view_response(response)
HTTP status code:
200
.... truncated for brevity ....
```
It is possible to use URI and query parameters concurrently in any call and add as many as the resource specifies.

If a resource requires a body to be supplied with data the body can be composed in the following way:

Check what the body of a call needs to look like by retrieving it out of the RAML file, 
and displaying it to stdout using ```view_resource_body_schema```:
```python
In [9]: client_session.view_resource_body_schema('logicalSwitches', 'create')

<virtualWireCreateSpec>
    <name>mandatory</name>
    <description/>
    <tenantId>mandatory</tenantId>
    <controlPlaneMode>mandatory</controlPlaneMode>
</virtualWireCreateSpec>
```
It is possible to create a template python dictionary using ```extract_resource_body_schema``` and display the output  structure in a human readable format to stdout:
```python
In [10]: new_ls = client_session.extract_resource_body_schema('logicalSwitches', 
															  'create')

In [11]: client_session.view_body_dict(new_ls)
{'virtualWireCreateSpec': {'controlPlaneMode': 'mandatory',
                           'description': None,
                           'name': 'mandatory',
                           'tenantId': 'mandatory'}}
```
It is possible to change any of the values in the dictionary with the data to be sent to the API:
```python
In [12]: new_ls['virtualWireCreateSpec']['controlPlaneMode'] = 'UNICAST_MODE'
In [13]: new_ls['virtualWireCreateSpec']['name'] = 'TestLogicalSwitch1'
In [14]: new_ls['virtualWireCreateSpec']['tenantId'] = 'Tenant1'

In [15]: client_session.view_body_dict(new_ls)
{'virtualWireCreateSpec': {'controlPlaneMode': 'UNICAST_MODE',
                           'description': None,
                           'name': 'TestLogicalSwitch1',
                           'tenantId': 'Tenant1'}}
```
This example shows how to send the call to the NSX Manager API by supplying the body dictionary in the call:
```python
In [16]: new_ls_response = client_session.create('logicalSwitches', 
												 uri_parameters={'scopeId': 
												                 'vdnscope-1'}, 
												 request_body_dict=new_ls)

In [17]: client_session.view_response(new_ls_response)
HTTP status code:
201

HTTP location header:
/api/2.0/vdn/virtualwires/virtualwire-1305

NSX Object Id:
virtualwire-1305

HTTP Body Content:
'virtualwire-1305'
```

### Note on Etag header and additional headers (e.g. If-match)

Some resources in NSX Manager will additionally need the ```If-match``` header.
To compose the ```If-match``` header, retrieve the content of the Etag and return it in the 
```If-match``` header. For example, this is used in the distributed firewall configuration to deal with
conflicts when multiple users try to concurrently edit rule sets.

This example shows how to retrieve a dfw rule, edit it, and update it via the NSX API:
```python
rule_read_response = client_session.read('dfwL3Rule', 
                                         uri_parameters={'sectionId': section_id,
                                                         'ruleId': new_rule_id})
updated_rule = l3_dfw_rule_read_response['body']
etag_value = l3_dfw_rule_read_response['Etag']

updated_rule['rule']['name'] = 'UpdatedByRAMLClient'

update_response = client_session.update('dfwL3Rule', 
                                        uri_parameters={'sectionId': section_id,
                                                        'ruleId': rule_id},
                                        additional_headers={'If-match': etag_value},
                                        request_body_dict=updated_rule)
```
Note that the ```If-match``` header is supplied by the ```additional_headers``` dictionary.

### Note on the use of XML Tags in body schemas

Some resources in NSX expect values to be set in XML Tags. This example shows a dfw resource:
```python
In [18]: client_session.view_resource_body_schema('dfwL3Rules', 'create')
<rule disabled="false" logged="false">
    <name>AddRuleTest</name>
    <action>allow</action>
    <notes/>
.... truncated for brevity ....
```
The ```rule```has the Tags ```disabled``` and ```logged```. When this type of Tag
is found, it is converted to a key prefixed by ```@``` in the resulting dictionary:
```python
In [19]: l3rule = client_session.extract_resource_body_schema('dfwL3Rules', 
                                                              'create')
In [20]: client_session.view_body_dict(l3rule)
{'rule': {'@disabled': 'false',
          '@logged': 'false',
          'action': 'allow',
.... truncated for brevity ....
```
It is possible to set values using the ```@``` prefix, and they will be converted to a XML Tag of the top level 
object.
```python
l3section_bdict['section']['rule'][0]['@logged'] = 'true'
```

### Note on repeating key/value pairs and resulting python lists containing dicts

In some cases NSX uses lists of parameters with repeating keys. For example:
```python
In [21]: client_session.view_resource_body_schema('dfwL3Section', 'create')
<section name="Test">
    <rule disabled="false" logged="true">
        <name/>
        <action>ALLOW</action>
        <appliedToList>
            <appliedTo>
                <name/>
                <value/>
                <type/>
                <isValid/>
            </appliedTo>
        </appliedToList>
        <sources excluded="false">
            <source>
                <name/>
                <value/>
                <type/>
                <isValid/>
            </source>
            <source>
                <name/>
                <value/>
                <type/>
                <isValid/>
            </source>
        </sources>
        <destinations excluded="false">
            <destination>
                <name/>
                <value/>
                <type/>
                <isValid/>
            </destination>
            <destination>
                <name/>
                <value/>
                <type/>
                <isValid/>
            </destination>
        </destinations>
        <services>
            <service>
                <destinationPort/>
                <protocol/>
                <subProtocol/>
            </service>
        </services>
    </rule>
    <rule disabled="false" logged="true">
       <name/>
       <action>DENY</action>
    </rule>
</section>
```
There are multiple ```destination``` keys under ```destinations```.
To be able to work with python dictionaries, nsxramlclient will convert those list of equally 
named parameter 'groups' to a Python list containing dictionaries.
This example shows the resulting Python dictionary for this type of resource body schema:
```python
In [22]: dfw_l3_sec = client_session.extract_resource_body_schema('dfwL3Section', 
                                                                  'create')
In [31]: client_session.view_body_dict(dfw_l3_sec)
{'section': {'@name': 'Test',
             'rule': [{'@disabled': 'false',
                       '@logged': 'true',
                       'action': 'ALLOW',
                       'appliedToList': {'appliedTo': {'isValid': None,
                                                       'name': None,
                                                       'type': None,
                                                       'value': None}},
                       'destinations': {'@excluded': 'false',
                                        'destination': [{'isValid': None,
                                                         'name': None,
                                                         'type': None,
                                                         'value': None},
                                                        {'isValid': None,
                                                         'name': None,
                                                         'type': None,
                                                         'value': None}]},
                       'name': None,
                       'services': {'service': {'destinationPort': None,
                                                'protocol': None,
                                                'subProtocol': None}},
                       'sources': {'@excluded': 'false',
                                   'source': [{'isValid': None,
                                               'name': None,
                                               'type': None,
                                               'value': None},
                                              {'isValid': None,
                                               'name': None,
                                               'type': None,
                                               'value': None}]}},
                      {'@disabled': 'false',
                       '@logged': 'true',
                       'action': 'DENY',
                       'name': None}]}}
```
Note the ```rule``` key, its value is a python List containing multiple rule objects that themselves
are python dictionaries. The same holds true for the ```destinations```and ```sources``` keys.

### License

Copyright © 2015 VMware, Inc. All Rights Reserved.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
documentation files (the "Software"), to deal in the Software without restriction, including without limitation
the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and
to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions
of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED
TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF
CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
IN THE SOFTWARE.

### How to contribute

Any contributions are welcome, bug reports, additional tests, enhancements, etc. 
Also we welcome your feedback if you find that anything is missing that would make nsxramlclient better
