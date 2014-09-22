Here is the blog article that explains the code sample.  Here is the url for the actual blog post http://kalho13.blogspot.com/#one

 Salesforce Mobile Single Page Application using the Force.com-JavaScript-REST-Toolkit
After many years of developing and customizing CRM Applications, I have now added Salesforce.com to my repertoire. In the nearly 3 years since, I have found Salesforce.com to be a flexible platform for solving problems by simply extending functionality on the existing model. As an example for this post I have a custom application within Salesforce which is used to help manage custom steel fabrication jobs, some weighing in excess of 100,000 pounds with hundreds and thousands of parts. Just imagine a giant erector set. The pain point experienced was in the packing of these jobs for shipment (shipped over 5 continents), things invariably got missed for a variety of reasons.

A Bill of Materials (BOM) for the job already existed in our custom Salesforce.com application, but shipping was relegated to a paper version, sometimes many pages long. The solution was to extend a small portion of the Salesforce.com custom application to shipping to enable tracking of exactly what was and was not loaded (and for management to review progress) for shipment. After some thought and research I decided to create a small Single Page Application (SPA) using a visualforce page with the Force.com JavaScript Rest Toolkit. Now for those of you saying oh I would of never done it that way, the good thing about Salesforce.com is that there are many ways. It was also an opportunity to learn the Force.com JavaScript Rest Toolkit and more about SPAs.

A key requirement was usability and performance as there are a many parts to load, and time is always of the essence. In keeping it simple the solution has just three screens:

    List of Open Jobs

    List of Parts for Selected Job
        Find the Part number by scrolling.
        Find the Part number by entering the first few characters in the filter.

    Mark as Loaded for Selected Part
        Leave the default full quantity or override with a partial quantity.

    Repeating Screen Two once the first part in the list has been loaded.
        Note the first Part is no longer in the list as the default full quantity was used.

In the end this is not exactly rocket science, but the implementation of this simple solutions helps manage a big problem. The mobile application is very quick, the user can use quickly filter down the long list of parts, and when the full quantity of the part has been marked as loaded the part disappears from the list. In the end the Parts list for a Job will be empty when the Parts are all loaded. Anything remaining in the list identifies something not accounted for in the load.

Enough about generalities, let's get down to the details and some code. To see the code in full follow this link to GitHub. The first step is to setup the Force.com JavaScript Rest Toolkit. It is available on here on Force.com-JavaScript-REST-Toolkit on GitHub. The setup (which is well explained in the GitHub repository) process requires a few steps to include:

    Add the forceTK component to Salesforce.com
    Add the forceTK Controller to Salesforce.com
    Add the forceTKControllerTest to Salesforce.com

The next thing we need is the Mobile Pack for jQuery which has all of the libraries and stylesheets required for the SPA. The jQuery Mobile pack is also available on GitHub. The code in this SPA is similar to the sample provided with the jQuery Mobile pack on Git Hub, but will extended functionality to include navigating to different menu levels. In this article I am going to reference and talk about key points in the code. For the full code detail use this GitHub link. To layout the flow of the SPA I will use some pseudo code first to demonstrate the flow of the logic.

    Instantiate the ForceTK Client
    Create objects for a Job, Part, and PartDetail
    Create click handler for button on PartDetail Page
    Query and Display List of Jobs
    Click and Job and Display Parts for the Job
    Click on Part and Display Part Detail
        Which only has quantity to load
    Click on Mark as Loaded Button
        Change part quantity if not all loaded.
    Redisplay the Parts for the Job
        The part quantity change reflected if not all were loaded
        The part no longer in the list if all were loaded


Initialize the ForceTK Client
While it is not readily apparent what the initialization process does, it is a step that you cannot skip.

 
    var $j = jQuery.noConflict(); 
    var client = new remotetk.Client();
    Force.init(null,null,client,null);

Create objects for a Job, Part, and PartDetail
Notice the Detail object is a little different as we are posting a this object record back to the Salesforce.com custom app when marking an item as loaded. The calculation of the remaining quantity to be loaded is based on the quantity on the Part (from the BOM) less the roll-up field of the child records for each time this part was loaded.

 

    var Jobs = new SObjectData();
    Jobs.errorHandler = displayError;

    var Parts = new SObjectData();
    Parts.errorHandler = displayError;

    var Details = new SObjectData('Job_Shipping_Details__c',['Id','Job_Part__c','Number_Loaded__c']);
    Details.errorHandler = displayError;

Create click handler for button on PartDetail Page
Using the jQuery ready event for the page, the click handler is created for the Save button (Label: Mark as Loaded).The click handler simply calls the updatePartRecord(event), which creates a child record to the parent Part in the custom Salesforce app.

$j(document).ready(function() {
   regBtnClickHandlers();
   getOpenJobs();
 });

  function regBtnClickHandlers() {                        
    $j('#save').click(function(e) {
     updatePartRecord(e);
    });
 }

Query and Display List of Jobs
Using the jQuery ready event for the page, the list of Jobs is loaded to the page using a simple query.

$j(document).ready(function() {
   regBtnClickHandlers();
   getOpenJobs();
 });

 function getOpenJobs() {
    Jobs.fetch('soql',"SELECT id, Name, Status__c FROM Job__c WHERE Status__c ='Open'",function() {
       showJobs(Jobs.data());
    });
 }

As we show the Jobs the code gets a little more complex as a link is added to query all of the parts for the selected job. This is where the code differentiates from the jQuery Mobile Pack example as this creates navigation through three levels of parent child records within Salesforce.com.

 function showJobs(records) {    
  $j('#cList').empty(); //make sure the list on the page has been cleared
   $j.each(Jobs.data(),
      function() {
        var newLi = $j('<li></li>');     //create a new list       
        var newLink = $j('<a id="' +this.Id+ '" onclick=getAllParts("'+this.Name+'"); data-transition="flip">[Job Name]: ' +this.Name+ '</a>'); //create a link variable
        newLink.click(function(e) {  //assign a link to the variable
          e.preventDefault();
          $j.mobile.showPageLoadingMsg();    //displays a loading image                 
          $j('#jobId').val(Jobs.findRecordById([this.id]).Id);
          $j('#jobName').val(Jobs.findRecordById([this.id]).Name);
          $j('#error').html('');    
        });

        newLi.append(newLink);       //append the link to the list     
        newLi.appendTo('#cList');   //append the list to the html spa list
   });//close each                
   $j.mobile.hidePageLoadingMsg(); //hides the loading image
   $j('#cList').listview('refresh'); //refresh to show all of the changes to the list
}  //close showJobs  

Click and Job and Display Parts for the Job
When the Job is selected, the getAllParts function is called, which loads all of parts for the job that meets the criteria of having a shipping load balance of greater than 0. When the parts lists loads the quantity to be loaded is also displayed along with the Part number.

 
  function getAllParts(name){
   var query = 'SELECT id, Name, Description__c, Shipping_Balance__c,Quantity__c FROM Job_Part__c WHERE Shipping_Balance__c > 0 and Shipping__c = true and Job__r.Name=' +"'"+ name +"'"+ "Order By Name ASC";
   Parts.fetch('soql',query,function() {
     showParts(Parts.data());
   });
 }

Once the Parts are queried, the showParts function is called and works much like the showJobs function which creates a list of Parts each with a link to the next level down in the record hierarchy.

 
 function showParts(records) {    
   $j('#pList').empty(); //clear the parts list
    $j.each(Parts.data(), //loop through each of the parts
        function() {
   var newLi = $j('<li></li>');                                
   var newLink = $j('<a id="' +this.Id+ '"data-transition="flip">' +this.Name+ '   :   (' + this.Shipping_Balance__c+')</a>');                     
          newLink.click(function(e) {
     e.preventDefault();
     $j.mobile.showPageLoadingMsg();
     $j('#shippingBalance').val(Parts.findRecordById([this.id]).Shipping_Balance__c);
     $j('#partId').val(Parts.findRecordById([this.id]).Id);
     $j('#partName').val(Parts.findRecordById([this.id]).Name);
     $j('#error').html('');                                          
     $j.mobile.changePage('#dialogpage', {changeHash: true});
          }); //end of the new link functionality
   
          newLi.append(newLink);            
   newLi.appendTo('#pList');
     }); //close each
            
     $j.mobile.hidePageLoadingMsg();
     $j('#partsHeader').text('Shipping Parts for ' + $j('#jobName').val());
     $j.mobile.changePage('#partspage',{changeHash:true});
     $j('#pList').listview('refresh');
  }     

Click on Part and Display Part Detail
In this instance a Part Detail record is written back to Salesforce.com, and new calculated is run on Salesforce (simply using the rollup field) and if there is a balance of greater than 0 to be loaded by shipping the Part remains in the list with a new quantity. If the remaining quantity to be loaded is 0 then the Part is no longer in the list.

 
  function updatePartRecord(e){
   e.preventDefault();
   var pId = $j('#partId').val();
   var sCount = $j('#shippingBalance').val(); get the quantity loaded from the 1 field form presented to the user.
   var record = Details.create('Job_Shipping_Details__c',{'Job_Part__c':pId,'Number_Loaded__c':sCount});
   Details.sync(record, successCallback); //with record successfully synced back to Salesforce the successCallback is called to load the list of Parts reflective of the change just made.
 }


Redisplay the Parts for the Job With record successfully synced back to Salesforce the successCallback is called to load the list of Parts reflective of the change just made.

 function successCallback(r){
  getAllParts($j('#jobName').val()); //make another call to getAllParts to reload the Parts list
  $j.mobile.changePage('#partspage', {changeHash: true});
 }

The SPA HTML
As a SPA all of the HTML is contained within a single page.  Throughout the code every reference to $j.mobile.changePage uses a portion of the code below rendering the three available pages, 'listpage', 'partspage', and 'dialogpage'.  The jQuery mobile and css magic takes over and does the rest with little effort of the part of the developer.  What a great library!

<body>   
    <div data-role="page" data-theme="b" id="listpage">                
      <div data-role="header" data-position="fixed">
        <h2>Jobs</h2>
      </div>
      <div data-role="content" id="jobList">            
        <ul id="cList" data-filter="true" data-inset="true" data-role="listview"  data-theme="c" data-dividertheme="b"></ul>
      </div>
    </div>
    <!--============== Parts Page ================= -->
    <div data-role="page" data-theme="b" id="partspage">                
      <div data-role="header" data-position="fixed">
        <h2 id="partsHeader"></h2>
        <a href='#listpage' id="add" class='ui-btn-right' data-icon='back' data-theme="b">Back</a>
    </div>
       <div data-role="content" id="partList">            
         <ul id="pList" data-filter="true" data-inset="true" data-role="listview"  data-theme="c" data-dividertheme="b"></ul>
      </div>
    </div>
    <!--============== Dialogbox ================= -->
 <div data-role="dialog" data-theme="b" id="dialogpage">
      <div data-role="header" data-position="fixed">
        <a href='#partspage' id="back2JobsList" class='ui-btn-right' data-icon='arrow-l' data-direction="reverse" data-transition="flip">Cancel</a>
        <h1>Shipping Load</h1>
      </div>
      <div data-role="content">
      <div data-role="fieldcontain">
     <label for="shippingBalance">Balance to be Loaded:</label>
     <input type="number" name="shippingBalance" id="shippingBalance" />
   </div>
   <h2 style="color:red" id="error"></h2><br/>
   <input type="hidden" id="jobId" />
         <input type="hidden" id="partId" />
         <input type="hidden" id="jobName" />
         <input type="hidden" id="partName" />
   <button id="save" data-role="button" data-icon="check" data-inline="true" data-theme="b" class="save">Mark As Loaded</button>
   </div>    
    </div>  
 </body>

In Conclusion
My objective in this blog was to provide an example of how I used a simple SPA, to extend an existing custom Salesforce.com app to solve a problem.  Hopefully this example will help you understand and create solutions of your own. 
==============
