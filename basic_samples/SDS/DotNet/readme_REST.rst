.NET Samples
============

Building a Client with the Sds REST API
--------------------------------------

This sample demonstrates how to interact with Sds using the Sds REST API. The REST API 
is language independent. Objects are passed as JSON strings. The sample uses the Newtonsoft.Json 
JSON framework, however, any method of creating a JSON representation of objects will work.

HTTP Client
-----------

The sample relies on the System.Net.Http.HttpClient to send and receive REST calls. The 
System.Net.Http.HttpClientFactory Create method is used to attach a 
System.Net.Http.DelegatingHandler that retrieves and attaches the authorization token to every message.

Authorization Handler
---------------------

The Sds Service is secured by Azure Active Directory. For a request to succeed, 
a valid token must be attached to every request sent to Sds. 

The sample includes a simple DelegatingHandler that relies on the 
Microsoft.IdentityModel.Clients.ActiveDirectory assembly to acquire the security token. 
The Authentication Handler accepts a resource, tenant, AAD instance format, 
client identifier and client secret. The handler supports an application identity.

Authentication-related values are received from OSIsoft. The values are provided to 
the sample in the appsettings.json configuration file as follows:

::

	{
		"NamespaceId": "REPLACE_WITH_NAMESPACE_ID",
		"TenantId": "REPLACE_WITH_TENANT_ID",
		"Address": "https://dat-a.osisoft.com",
		"Resource": "https://qihomeprod.onmicrosoft.com/ocsapi",
		"ClientId": "REPLACE_WITH_CLIENT_IDENTIFIER",
		"ClientKey": "REPLACE_WITH_CLIENT_SECRET",
		"ApiVersion": "v1-preview",
		"AADInstanceFormat": "https://login.windows.net/<REPLACE_WITH_TENANT_ID>/oauth2/token"
	}


The security handler is attached to the HttpClient as follows:

.. code:: cs

	SdsSecurityHandler securityHandler =
		new SdsSecurityHandler(resource, tenantId, aadInstanceFormat, clientId, clientKey);
			HttpClient httpClient = new HttpClient(securityHandler)
			{
				BaseAddress = new Uri(address)
			};
            
Note that Sds returns a status of 302 (Found), when metadata collisions exist. The HttpClient 
auto-redirect, which automatically issues a GET when receiving a 302, will result in an 
unauthorized response. Because HttpClient does not retain the authorization token on a redirect, 
it is recommended that auto redirect be disabled.


Create an SdsType
---------------

To use Sds, you define SdsTypes that describe the kinds of data you want
to store in SdsStreams. SdsTypes are the model that define SdsStreams.
SdsTypes can define simple atomic types, such as integers, floats, or
strings, or they can define complex types by grouping other SdsTypes. For
more information about SdsTypes, refer to the `Sds
documentation <https://cloud.osisoft.com/documentation>`__.

In the sample code, the SdsType representing WaveData is defined in the BuildWaveDataType
method of Program.cs. WaveData contains properties of integer and double atomic types. 
The constructions begins by defining a base SdsType for each atomic type and then defining
Properties of those atomic types.

.. code:: cs

	SdsType intSdsType = new SdsType
	{
		Id = "intSdsType",
		SdsTypeCode = SdsTypeCode.Int32
	};

	SdsType doubleSdsType = new SdsType
	{
		Id = "doubleSdsType",
		SdsTypeCode = SdsTypeCode.Double
	};

	SdsTypeProperty orderProperty = new SdsTypeProperty
	{
		Id = "Order",
		SdsType = intSdsType,
		IsKey = true
	};
	
	SdsTypeProperty tauProperty = new SdsTypeProperty
	{
		Id = "Tau",
		SdsType = doubleSdsType
	};

These properties are assembled into a collection and assigned to the Properties 
property of a new SdsType object:

.. code:: cs

	SdsType waveType = new SdsType
	{
		Id = id,
		Name = "WaveData",
		Properties = new List<SdsTypeProperty>
		{
			orderProperty,
			tauProperty,
			radiansProperty,
			sinProperty,
			cosProperty,
			tanProperty,
			sinhProperty,
			coshProperty,
			tanhProperty
		},
		SdsTypeCode = SdsTypeCode.Object
	};

Finally, the new SdsType object is submitted to the Sds Service:

.. code:: cs

	HttpResponseMessage response =
	await httpClient.PostAsync($"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Types/{waveType.Id}",
		new StringContent(JsonConvert.SerializeObject(waveType)));


Create an SdsStream
-----------------

An ordered series of events is stored in an SdsStream. All you have to do
is create a local SdsStream instance, give it an Id, assign it a type,
and submit it to the Sds service. You may optionally assign a
SdsStreamBehavior to the stream. The value of the ``TypeId`` property is
the value of the SdsType ``Id`` property.

.. code:: cs

	SdsStream waveStream = new SdsStream
	{
		Id = StreamId,
		Name = "WaveStream",
		TypeId = waveType.Id
	};


The local SdsStream can be created in the Sds service by a POST request as
follows:

.. code:: cs
	
	response = await httpClient.PostAsync($"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}",
		new StringContent(JsonConvert.SerializeObject(waveStream)));


Create and Insert Values into the Stream
----------------------------------------

A single event is a data point in the stream. An event object cannot be
empty and should have at least the key value of the Sds type for the
event. Events are passed in json format.

An event can be created using the following POST request:

.. code:: cs

	response = await httpClient.PostAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/InsertValue",
			new StringContent(JsonConvert.SerializeObject(wave)));


Inserting multiple values is similar, but the payload has list of events
and the url for POST call varies:

.. code:: cs

	List<WaveData> waves = new List<WaveData>();
	for (int i = 2; i < 20; i += 2)
	{
		WaveData newEvent = GetWave(i, 2, 2.0);
		waves.Add(newEvent);
	}
	response = await httpClient.PostAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/InsertValues",
			new StringContent(JsonConvert.SerializeObject(waves)));

The Sds REST API provides many more types of data insertion calls beyond
those demonstrated in this application. Go to the 
Sds documentation<https://cloud.osisoft.com/documentation> for more information
on available REST API calls.

Retrieve Values from a Stream
-----------------------------

There are many methods in the Sds REST API allowing for the retrieval of
events from a stream. The retrieval methods take string type start and
end values; in the case of the GetWindowValues call, these are the start and 
end ordinal indices expressed as strings. The index values must capable of 
conversion to the type of the index assigned in the SdsType.

.. code:: cs

	response = await httpClient.GetAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/GetWindowValues?startIndex=0&endIndex={waves[waves.Count - 1].Order}");

-  parameters are the SdsStream Id and the starting and ending index
   values for the desired window Ex: For a time index, request url
   format will be
   "/{streamId}/Data/GetWindowValues?startIndex={startTime}&endIndex={endTime}

As with data insertion, the Sds REST API provides many more types of data retrieval calls beyond
those demonstrated in this application. Go to the 
Sds documentation<https://cloud.osisoft.com/documentation> for more information
on available REST API calls.

Update Events and Replacing Values
----------------------------------

Updating events is handled by PUT REST call as follows:

.. code:: cs

	response = await httpClient.PutAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/UpdateValue",
			new StringContent(JsonConvert.SerializeObject(updateEvent)));

-  the request body has the new event that will update an existing event
   at the same index

Updating multiple events is similar, but the payload has an array of
event objects and url for PUT is slightly different:

.. code:: cs

	List<WaveData> updateWaves = new List<WaveData>();
	for (int i = 0; i < 40; i += 2)
	{
		WaveData newEvent = GetWave(i, 4, 6.0);
		updateWaves.Add(newEvent);
	}

	response = await httpClient.PutAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/UpdateValues",
			new StringContent(JsonConvert.SerializeObject(updateWaves)));

If you attempt to update values that do not exist they will be created. The sample updates
the original ten values and then adds another ten values by updating with a
collection of twenty values.

In contrast to updating, replacing a value only considers existing
values and will not insert any new values into the stream. The sample
program demonstrates this by replacing all twenty values. The calling conventions are
identical to ``updateValue`` and ``updateValues``:

.. code:: cs

	response = await httpClient.PutAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/ReplaceValue",
			new StringContent(JsonConvert.SerializeObject(replaceEvent)));

	response = await httpClient.PutAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/ReplaceValues",
			new StringContent(JsonConvert.SerializeObject(replaceEvents)));

Property Overrides
------------------

Sds has the ability to override certain aspects of an Sds Type at the Sds Stream level.  
Meaning we apply a change to a specific Sds Stream without changing the Sds Type or the
behavior of any other Sds Streams based on that type.  

In the sample, the InterpolationMode is overridden to a value of Discrete for the property Radians. 
Now if a requested index does not correspond to a real value in the stream then ``null``, 
or the default value for the data type, is returned by the Sds Service. 
The following shows how this is done in the code:

.. code:: cs

	// Create a Discrete stream PropertyOverride indicating that we do not want Sds to calculate a value for Radians and update our stream
	SdsStreamPropertyOverride propertyOverride = new SdsStreamPropertyOverride
	{
		SdsTypePropertyId = "Radians",
		InterpolationMode = SdsInterpolationMode.Discrete
	};

	var propertyOverrides = new List<SdsStreamPropertyOverride>() { propertyOverride };

	// update the stream
	waveStream.PropertyOverrides = propertyOverrides;
	response = await httpClient.PutAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}",
			new StringContent(JsonConvert.SerializeObject(waveStream)));

The process consists of two steps. First, the Property Override must be created, then the
stream must be updated. Note that the sample retrieves three data points
before and after updating the stream to show that it has changed. See
the `Sds documentation <https://cloud.osisoft.com/documentation>`__ for
more information about Sds Property Overrides.


SdsStreamViews
-------

An SdsStreamView provides a way to map Stream data requests from one data type 
to another. You can apply a StreamView to any read or GET operation. SdsStreamView 
is used to specify the mapping between source and target types.

Sds attempts to determine how to map Properties from the source to the 
destination. When the mapping is straightforward, such as when 
the properties are in the same position and of the same data type, 
or when the properties have the same name, Sds will map the properties automatically.

.. code:: cs

	response =
		await httpClient.PostAsync($"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/StreamViews/{AutoStreamViewId}",
			new StringContent(JsonConvert.SerializeObject(autoStreamView)));

To map a property that is beyond the ability of Sds to map on its own, 
you should define an SdsStreamViewProperty and add it to the SdsStreamView's Properties collection.

.. code:: cs

	// create explicit mappings 
	var vp1 = new SdsStreamViewProperty() { SourceId = "Order", TargetId = "OrderTarget" };
	var vp2 = new SdsStreamViewProperty() { SourceId = "Sin", TargetId = "SinInt" };
	var vp3 = new SdsStreamViewProperty() { SourceId = "Cos", TargetId = "CosInt" };
	var vp4 = new SdsStreamViewProperty() { SourceId = "Tan", TargetId = "TanInt" };

	var manualStreamView = new SdsStreamView()
	{
		Id = ManualStreamViewId,
		SourceTypeId = TypeId,
		TargetTypeId = TargetIntTypeId,
		Properties = new List<SdsStreamViewProperty>() { vp1, vp2, vp3, vp4 }
	};

SdsStreamViewMap
---------

When an SdsStreamView is added, Sds defines a plan mapping. Plan details are retrieved as an SdsStreamViewMap. 
The SdsStreamViewMap provides a detailed Property-by-Property definition of the mapping.
The SdsStreamViewMap cannot be written, it can only be retrieved from Sds.

.. code:: cs

	response = await httpClient.GetAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/StreamViews/{AutoStreamViewId}/Map");     

Delete Values from a Stream
---------------------------

There are two methods in the sample that illustrate removing values from
a stream of data. The first method deletes only a single value. The second method 
removes a window of values, much like retrieving a window of values.
Removing values depends on the value's key type ID value. If a match is
found within the stream, then that value will be removed. Code from both functions
is shown below:

.. code:: cs

	response = await httpClient.DeleteAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/RemoveValue?index=0");

	response = await httpClient.DeleteAsync(
		$"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{waveStream.Id}/Data/RemoveWindowValues?startIndex=0&endIndex=40");

As when retrieving a window of values, removing a window is
inclusive; that is, both values corresponding to '0' and '40'
are removed from the stream.

Cleanup: Deleting Types, Behaviors, StreamViews and Streams
-----------------------------------------------------

In order for the program to run repeatedly without collisions, the sample
performs some cleanup before exiting. Deleting streams, stream
behaviors, streamViews and types can be achieved by a DELETE REST call and passing
the corresponding Id.

.. code:: cs

	await httpClient.DeleteAsync($"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Streams/{StreamId}");

.. code:: cs

	await httpClient.DeleteAsync($"api/{apiVersion}/Tenants/{tenantId}/Namespaces/{namespaceId}/Types/{TypeId}");
