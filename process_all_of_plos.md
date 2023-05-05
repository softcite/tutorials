# Tutorial: process all of PLOS articles

This is a tutorial to illustrate how to process efficiently the PLOS open access collection of articles in XML format. 

The full collection is available [here](https://allof.plos.org/allofplos.zip), updated every days. You can also use <https://github.com/PLOS/allofplos> to retrieve new publications on a regular basis, e.g. daily incremental update. 

As for 2023-05-05, the corpus contains 336,168 files (7.5G compressed, 38G uncompressed). 

Note: With this archive, all the XML files are written under the same directory. This is a lot of files under the same directory and this is known to be inefficient in term of file system (we usually distribute files in smaller subdirectories of a few hundred files maximum). However, the number of files here is still manageable, with a relatively limited runtime immpact. For the sake of simplicity, we do not consider re-organizing these files on the file system, but this could be necessary as the corpus is growing. 

## Install the Softcite software mention recognizer service, the client and Pub2TEI

#### 1) Softcite software mention recognizer service

For using the Softcite software mention recognizer, a Docker image is available containing all the Deep LEarning models with the best accuracy settings. It will recognize automatically available GPU. 

```console
docker pull grobid/software-mentions:0.7.3-SNAPSHOT
docker run --rm --gpus all -it -p 8060:8060 grobid/software-mentions:0.7.3-SNAPSHOT
```

The service is running on the port `8060`. 

#### 2) Python Softcite recognizer client

Install the python client as follow: 

```console
> git clone https://github.com/softcite/software_mentions_client.git
> cd software_mentions_client/
```

It is advised to setup first a virtual environment to avoid falling into one of these gloomy python dependency marshlands:

```console
> virtualenv --system-site-packages -p python3 env
> source env/bin/activate
```

Install the dependencies, use:

```console
> python3 -m pip install -r requirements.txt
```

Finally install the project in editable state

```console
> python3 -m pip install -e .
```

#### 3) Pub2TEI

To install Pub2TEI:

```console
git clone https://github.com/kermitt2/Pub2TEI
```

To use Pub2TEI, you will need Java installed on your system. Any Java versions should work. 

## Transforming the XML JATS documents into TEI

The Softcite software mention recognizer can process directly JATS XML files. For example: 

```
curl --form input=@/media/lopez/data/allofplos/journal.pone.0124721.xml --form disambiguate=1 localhost:8060/service/extractSoftwareXML
```

However when processing a large amount of JATS files, this is inefficient because for each call a Pub2TEI transformation is realized from scratch by the server, which means loading and compiling the full set of XSLT stylesheets, which takes around 2 seconds each time. 

A more efficient approach is to transform first all the XML JATS files into XML TEI in a batch application of Pub2TEI and then send TEI files to the service. With the batch approach, as the XSLT stylesheets are loaded and compiled only time for all the processed JATS file, it should lead to the transformation of 50-100 JATS files per second. Then the service will process TEI files without any pre-processing, which will be much faster. 

For batch conversion of JATS files under `/media/lopez/data/allofplos` with TEI files written under `/media/lopez/data/allofplos/tei`, move under the Pub2TEI install repository and use:

```console
cd ~/Pub2TEI
java -jar Samples/saxon9he.jar -s:/media/lopez/data/allofplos -xsl:Stylesheets/Publishers.xsl -o:/media/lopez/data/allofplos/tei -dtd:off -a:off -expand:off -t --parserFeature?uri=http%3A//apache.org/xml/features/nonvalidating/load-external-dtd:false 
```

After 2 to 3 hours, depending on your hardware, all the JATS files will be converted into TEI files. As it is a single thread process, it is possible to speed-up significantly this step by parallelizing the transformations, but likely not worth the effort (longer coffee breaks/dog walks are always welcome!). 

## Processing TEI files

Simply use the client to process all the TEI files in parallel passing the repository path:

```console
cd software_mentions_client
python3 -m software_mentions_client.client --repo-in /media/lopez/data/allofplos/tei 
```

Be sure to have the correct URL of the softcite server in the `config.json` of the client. 

The extracted software mentions will be written in JSON files along the XML files. The process can be interrupted and resumed. 

On a 5 years old desktop machine, with nvidia GTX 1080Ti GPU (11 GB memory), the server process around 1 TEI file per second (as a comparison with a similar parallel processing mode, one PDF file is processed in average in 2 seconds). 

We processed the full PLOS XML corpus in slightly more than **4 days with one server**. 

## Annex: Processing XML JATS or PDF? 

For the PLOS collection, both JATS and PDF formats are available in open access. 

Processing JATS has several advantages:

* The text has higher quality, because the PDF extraction contains a certain amount of noise and mistakes, failing UTF-8 code resolution, issues with embedded fonts, etc..

* The document structures are more reliable than from the ones from the automatic PDF structuring with GROBID. Working with PDF might result in skipping some paragraphs badly recognized or to process unrelevant text pieces - although not very frequent overall, these errors can impact the quality of software mention extraction by some few pourcents.  

* As indicated above, processing XML files is two times faster than processing a PDF, because of the required PDF parsing and automatic structuring.

The drawback of processing XML files is that there is no coordinates of mention elements extracted, which can be useful for some partiicular application/use cases:

- It is not possible to visualize the annotations on the PDF and to offer interactive layout based on these extractions. 

- Coordinates offer convenient canonical identification of annotations for a given PDF. If new annotations are produced, or some annotations are corrected manually by humans, they can be matched and compared with existing annotations based on their coordinates. 

As a conclusion, processing JATS documents should be preferable in most of the use cases. 
