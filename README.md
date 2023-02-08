# How to Identify UBI Base Images
Once upon a time there is an use-case where we have to identify more 100 container images that is a RedHat UBI based or 3rd party container images.
Let say we want to find out if there are any Non-UBI based images e.g. 3rd party images are used in Telco CNFs that stored on RedHat private Quay registry server, we can manual click on each image or use some tools to inspect them manual to figure it out each of them is an UBI based image. Either way it takes a lot of times to identify it one by one. 

That's why I write this knowledge base to simplify and automate using shellscript and REST API to identify and generate results into CSV files or on the screen without 2-3 minutes for 100 images. 
