---
layout: post
title: Using Google Charts With AngularJs
date: '2013-07-30 09:28:16'
---

<p>I recently rewrote a small webpage in AngularJs and it was amazing how much lighter and cleaner it made the code. The conversion was mainly a case of me throwing away most of the code and adding a couple of lines of AngularJs. This was all very simple until I got to some Google Charts, at this point I had the option of leaving them as they were or converting them into AngularJs directives. I went with the directives option to keep everything in one place and work with the existing models I already have in my scope.</p> <p>The next decision was whether or not to have the charts&nbsp; automatically re-render when the model changes. In this case I chose not to do that as I decided flashing charts every time the model changes would be annoying to the user. If you did however choose to go this route it should be quite simple using $watch on your datatables to tell the graph when to redraw.</p> <p>Take this simple page as an example</p>

```language-markup
<!DOCTYPE html>
<html lang="en">
    <head>
         <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.7/angular.js" type="text/javascript"></script>
         <script src="https://www.google.com/jsapi" type="text/javascript"></script>
         <script src="main.js" type="text/javascript"></script>
         <script src="ngGoogleCharts.js" type="text/javascript"></script>
         <style type="text/css">
             .bigGraph {width:500px;height:500px;float:left;}
             .mediumGraph {width:400px;height:400px;float:left;}
             .smallGraph {width:200px;height:200px;float:left;}
         </style>
    </head>
    <body ng-controller="IndexCtrl">
        <div google-chart="PieChart" ng-model="data1" class="bigGraph"></div>
        <div google-chart="BarChart" ng-model="data2" class="mediumGraph"></div>
        <div google-chart="LineChart" ng-model="data3" class="smallGraph"></div>
    </body>
</html>
```
<p>Hopefully this looks fairly simple. If you’ve used AngularJs before you might notice the absence of ng-app, this reason for this is if Angular runs before the Google Charts have loaded nothing they wont render. So I’ve had to&nbsp; manually instantiate the Angular app later once Google Charts are ready. </p>
<p>I put the main Angular module in main.js and it’s in here where I instantiate Angular and create some hard coded datatables for the charts to run on.</p>

```language-javascript
 "use strict";

/*We need to manually start angular as we need to
wait for the google charting libs to be ready*/
google.setOnLoadCallback(function () {    
    angular.bootstrap(document.body, ['my-app']);
});
google.load('visualization', '1', {packages: ['corechart']});


var myApp = myApp || angular.module("my-app",["google-chart"]);

myApp.controller("IndexCtrl",function($scope){    
    $scope.data1 = {};
    $scope.data1.dataTable = new google.visualization.DataTable();
    $scope.data1.dataTable.addColumn("string","Name")
    $scope.data1.dataTable.addColumn("number","Qty")
    $scope.data1.dataTable.addRow(["Test",1]);
    $scope.data1.dataTable.addRow(["Test2",2]);
    $scope.data1.dataTable.addRow(["Test3",3]);
    $scope.data1.title="My Pie"

    $scope.data2 = {};
    $scope.data2.dataTable = new google.visualization.DataTable();
    $scope.data2.dataTable.addColumn("string","Name")
    $scope.data2.dataTable.addColumn("number","Qty")
    $scope.data2.dataTable.addRow(["Test",1]);
    $scope.data2.dataTable.addRow(["Test2",2]);
    $scope.data2.dataTable.addRow(["Test3",3]);


    $scope.data3 = {};
    $scope.data3.dataTable = new google.visualization.DataTable();
    $scope.data3.dataTable.addColumn("string","Name")
    $scope.data3.dataTable.addColumn("number","Qty")
    $scope.data3.dataTable.addRow(["Test",1]);
    $scope.data3.dataTable.addRow(["Test2",2]);
    $scope.data3.dataTable.addRow(["Test3",3]);
});
```
<p>The next step was to add the logic that knows what to do with the “google-chart” attributes on the charting divs in the view. This is where Angular directives come in. I created a directive that matches on this attribute and creates the chart. In this case I put the directive in its own Angular module in a file called ngGoogleCharts.js</p>

```language-javascript
"use strict";

var googleChart = googleChart || angular.module("google-chart",[]);

googleChart.directive("googleChart",function(){
    return{
        restrict : "A",
        link: function($scope, $elem, $attr){
            var dt = $scope[$attr.ngModel].dataTable;
    
            var options = {};
            if($scope[$attr.ngModel].title)
                options.title = $scope[$attr.ngModel].title;

            var googleChart = new google.visualization[$attr.googleChart]($elem[0]);
            googleChart.draw(dt,options)
        }
    }
});
```
<p>It’s a very simple example but should serve as a good starting point for using AngularJs directives to render Google Charts. </p>