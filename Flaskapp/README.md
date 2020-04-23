## HELM CUSTOMISED CHART


For creating a chart
  STEPS:

     helm create <chart_name>
     ls
     vi values.yaml(edit this file according to your needs)
 also look into the templates:
     cd templates
 edit the default files as per your need

To verify the chart

    helm lint


For installing the chart

  STEPS:
   
     helm install <chart_name> ./<chart_path/directory>
       
