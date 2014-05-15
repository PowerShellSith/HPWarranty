This module contains four functions:
Invoke-SOAPRequest
Invoke-HPWarrantyRegistrationRequest
Invoke-HPWarrantyLookup
Get-HPComputerInformationForWarrantyRequestFromCCMDB

The Invoke-SOAPRequst is not really used by a user, but it is used by the two other Invoke-HP* functions.  Basically, to use HPs ISEE to get warranty info, there are a few things that need to happen:

1.  A Session needs to be established with their Web Services, this is done by using the Invoke-HPWarrantyRegistrationRequest.  You need a valid SerialNumber and Product Model to do this.  If successful, it will return a Gdid and a Session Token, that then needs to be passed to the Invoke-HPWarrantyLookup.
2.  Using the Gdid and the Token from step 1, and a valid SerialNumber and ProductNumber you can request the information for that device, returning the SerialNumber,WarrantyStartDate,WarrantyStandardEndDate, and WarrantyExtendedEndDate.  (The return SOAP Envelop contains much more information, but that is was I parse and return).
3.  You can reuse the Gdid and Token from step one in a foreach loop to retrieve multiple warranty objects.  That is why I created the Get-HPComputerInformationForWarrantyRequestFromCCMDB.  If you configure your SCCM Client to inventory the MS_SystemInformation WMI Class (Found in the root namespace, not in CIMV2, root\MS_SystemInformation) you can then use this function to return form the CM_<SiteCode> database an array of objects containing the information needed to complete all actions.  SerialNumber,ProductModel,ProductNumber,ProductManufacturer.

 

Examples 1:

	# Execute from a local HP Workstation
	Import-Module -Name HPWarranty; $reg = Invoke-HPWarrantyRegistrationRequest; Invoke-HPWarrantyLookup -Gdid $reg.Gdid -Token $reg.Token

Example 2:

	# Create one session but look up multiple warranty's
	Import-Module HPWarranty
	
	$HP1 = @{
		'SerialNumber' = 'A1B2C3D4E5'
		'ProductModel' = 'HP Laptop 100 G1'
		'ProductNumber' = '123ABC'
	}
	
	$HP2 = @{
		'SerialNumber' = '12345ABCDE'
		'ProductModel' = 'HP Desktop 1100 G1'
		'ProductNumber' = 'ABC123'
	}
	
	# Use either HP1 or HP2 properties to establish a session with the HP Web Services.
	$reg = Invoke-HPWarrantyRegistrationRequest -SerialNumber $HP1.SerialNumber -ProductModel $HP1.ProductModel
	
	Invoke-HPWarrantyLookup -Gdid $reg.Gdid -Token $reg.Token -SerialNumber $HP1.SerialNumber -ProductNumber $HP1.ProductNumber
	Invoke-HPWarrantyLookup -Gdid $reg.Gdid -Token $reg.Token -SerialNumber $HP2.SerialNumber -ProductNumber $HP2.ProductNumber

Example 3:

	# Query a remote computer for information to create a session with the the HP Web Services.
	# Remote WMI access is necessary to use this function.
	Import-Module -Name HPWarranty
	
	$reg = Invoke-HPWarrantyRegistrationRequest -ComputerName HPComputer.mydomain.org
	
	Invoke-HPWarrantyLookup -Gdid $reg.Gdid -Token $reg.Token -ComputerName HPComputer.mydomain.org
	
Example 4:

	# Execute with information from ConfigMgr Database:
	Import-Module -Name HPWarranty

	$reg = Invoke-HPWarrantyRegistrationRequest -SeralNumber "ABCDE12345" -ProductModel "HP ProBook 645 G1"

	$HPs = Get-HPComputerInformationForWarrantyRequestFromCCMDB -Server MySccmDBServer -Database CM_MS1 -IntergratedSecurity
	foreach ($HP in $HPs)
	{
		 Invoke-HPWarrantyLookup -Gdid $reg.Gdid -Token $reg.Token -SerialNumber $HP.SerialNumber -ProductNumber $HP.ProductNumber
	}