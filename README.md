# Neural Web
This API provides
1. tagging with Neural Networks for INCEpTION (as external recommender)
2. automatic feedback generation for CASUS using tagging with Neural Networks for EDA feedback and string-matching for content feedback

Currently the Neural Network used is a BiLSTM-CRF by Nils Reimers implemented in Keras and using the Theano backend.

However, any other network can be used by creating a class \<yournetwork>Wrapper that inherits from your network and the abstract class `neuralNets/network.py`.
Use `neuralNets/BiLSTMWrapper.py` as an example.

---
## API for INCEpTION

neural-web receives a JSON for tagging that looks like this:
```json
{
      "metadata": {
        "layer": "LayerName_String",
        "feature": "FatureOfTheLayer_String",
        "projectId": "InceptionProjectID_Integer",
        "anchoringMode": "InceptionAchoringMode_String",
        "crossSentence": "IsCrossSentenceTokenAllowed_Boolean"
      },
      "document": {
        "xmi": "EscapedPlainTextXMI_XMLString",
        "documentId": "InceptionDocumentID_Integer",
        "userId": "InceptionUserID_String"
      },
      "typeSystem": "EscapedPlainTextXMI_XMLString"
}
```

and for training JSON is very similar but looks like this:
```json
{
      "metadata": {
        "layer": "LayerName_String",
        "feature": "FatureOfTheLayer_String",
        "projectId": "InceptionProjectID_Integer",
        "anchoringMode": "InceptionAchoringMode_String",
        "crossSentence": "IsCrossSentenceTokenAllowed_Boolean"
      },
      "documents": [
          {
            "xmi": "EscapedPlainTextXMI_XMLString",
            "documentId": "InceptionDocumentID_Integer",
            "userId": "InceptionUserID_String"
          }
      ],
      "typeSystem": "EscapedPlainTextXMI_XMLString"
}
```

---
## API for CASUS
neural-web receives a JSON for tagging that looks like this:
```json
{
    "text": "<BASE64Encoded_UTF8String>",
    "user":"<String_IDofUser>",
    "case": "<String_numberOfCase>",
    "api-key": "<givenToClientByUs>",
    "proof": "<blake2bEncodingOfText>"
}
```

No training facility for CASUS implemented yet.

# Setup

## Setup nltk and punkt

This needs to be done only once and may be ignored if nltk is already used previously with the system.

To install NLTK:

```
sudo pip3 install -U nltk
```

Once nltk is installed open terminal and go to the python3 shell, then use:

```
import nltk
nltk.download('punkt')
nltk.download('stopwords')

from nltk import word_tokenize,sent_tokenize
```

The last import command verifies if everything is installed correctly.

---
## Setup with virtual environment

Setup a Python virtual environment:
```
virtualenv env --python=python3.5
source env/bin/activate
```

Install the requirements:
```
python -m pip install -r requirements.txt
```

If everything works well, you can run `python app.py ${config}` to start the Server, `${config}` can be filled with values `dev` or `prod`

---
## Build and run docker image

Right now neural web has two versions hosted on server. One for production and the other for development, using ports `12889` and `12890` respectively.  

### Building and hosting production server

Build the docker image from the root folder of the repository. Run:
```
sudo docker build ./docker/ -f ./docker/Dockerfile -t neural-web-prod
```

This builds a Python 3.5 image with additional dependencies form `docker/requirements.txt` installed and assigns the name *neural-web-prod* to it.

To run our code, we first must create a container form the previously created image and start the container and mount the current folder ${PWD} into the container:

* `docker run ...` command first creates a container using `docker create` and then starts it using `docker start`
```
sudo docker run --restart on-failure -d -it -v ${PWD}:/usr/src/app -p 12889:12889 -e host_config=prod neural-web-prod
```

The command `-v ${PWD}:/usr/src/app` maps the current folder ${PWD} into the docker container at the position `/user/src/app`. Changes made on the host system as well as in the container are synchronized. We can change / add / delete files in the current folder and its subfolder and can access those files directly in the docker container.

*Here `-p host-port:conatiner-port`. Please select these two ports according to your `util/config.py`*

app.py will be executed automatically when the container has started.

### Building and hosting dev server

All the previous constraints also apply to dev server.

To Build the image run:

```
sudo docker build ./docker/ -f ./docker/Dockerfile -t neural-web-dev
```

For creating and hosting use: 
```
sudo docker run --restart on-failure -d -it -v ${PWD}:/usr/src/app -p 12890:12890 -e host_config=dev neural-web-dev
``` 
### Notes on rebuilding and/or re-hosting
To rebuild and re-host:
* **This should be necessary only when we have changed something in requirements.txt and/or Dockerfile**
1. Find the desired container using `docker ps -a`; will print a list of all running and stopped container.
2. Use `docker stop <param>`; `<param>` might be either the CONTAINER ID or NAME
3. Then remove the container with `docker rm <param>`; again `<param>` might be either the CONTAINER ID or NAME
4. Then remove the associated image using `docker rmi <param>`; now this will be image name i.e. `neural-web-dev` or `neural-web-prod`
5. Change source files (i.e. editing, pulling etc.)
6. Build the image using method stated above; use `docker build ...` .
8. Create and run the conatiner with changed source as per above using `docker run ...`

To only re-host:
* **This should be sufficient only for source code modification**
1. Find the desired container using `docker ps -a`; will print a list of all running and stopped container.
2. Use `docker stop <param>`; `<param>` might be either the CONTAINER ID or NAME
3. Change source files (i.e. editing, pulling etc.)
4. Start the container using `docker start <param>`; again `<param>` might be either the CONTAINER ID or NAME

 *CONTAINER NAME is given automatically by docker when a new container is created and multiple container can be created using same image*


---
## Add trained Neural Network models

Paste your model(s) in the folder `models`.


# API Functionality

## Load a model
By defeault, all models in the  `models` folder are loaded upon starting the Server.
To load a model that was not in the folder, add it to the folder and send a POST request to `localhost:12889/loadModel/<modelName>` and wait for the response.
`<modelName>` is the name of the model without the `.h5` extension.

---
## Tag a Document sent as CAS-XMI (for INCEpTION)

To tag a document with the default model send a POST request to `localhost:12890/api/inception/v1/tag`. Provide a JSON with the document in the body of the request as specified above. 
An appropriate `model` will be automatically chosen for tagging from request `json`. If the determined model is not found will reply with a HTTP 404 response. 

---
## Train a model from Documents sent as CAS-XMI (for INCEpTION)

To train a model using some previously tagged and completed documents send a POST request to `localhost:12890/api/inception/v1/tag`. Provide a JSON with the documents in the body of the request as specified above. 
An appropriate `model` will be automatically chosen for tagging from request `json`. If the determined model is not found this will build and train a new model otherwise re-train the existing model.

**Notes :**
* To start training this system needs at least 30 new documents. Can be configured in `config.py` using `training_config.training_document_threshold` key.
* No no need to re-send previously sent but documents that did not took part in training.

---
## Tag a Document sent as String (for CASUS or other applications)

To tag a document with the default model send a POST request to `localhost:12889/api/external/v1/tag`. Provide a JSON with the document in the body of the request as specicified above, but without the `case` and `user` fields. 
To tag a document using a specific model send the POST request to `localhost:12889/api/external/v1/tag/<modelName>`. `<modelName>` is the name of the model without the `.h5` extension.

---
## Get automatic feedback for a Document sent as String (for CASUS)

To receive feedback regarding argument structure (using the default model - medicine) and the content of a text (default content currently medicine), send a POST request to `localhost:12889/api/external/v1/feedback`. Provide a JSON with the document in the body of the request as specicified above.
To receive feedback regarding argument structure using a specific model and the content of a text (default content currently medicine), send a POST request to `localhost:12889/api/external/v1/feedback/<modelName>`. Provide a JSON with the document in the body of the request as specicified above.

# Example scripts for minimal clients

* `client_inception.py` - An example of expected inception tagging request.
* `client_inception_train.py` - An example of expected inception training request.
* `client_external_partner_feedback.py` - An example of expected tagging request from external partners to provide them with feed back.
* `client_external_partner_tagging.py` - An example of expected external partner's tagging request.
* `IOB_to_CAS_file_generartor.py` - An example of transforming IOB tagged files into CAS XMI. Some example of IOB files used are form [Neural-Web Training Experiment project](https://git.ukp.informatik.tu-darmstadt.de/ibna/neural-web-train-exp/tree/master/trainDocuments/).


# Web-Frontend for automatic feedback (for Flair)

Steps:

1. Start backend Flair-Server from the project directory: 

    `python3 app.py [config]` 
    
    By default, the address and port set in util.config.py will be used. 
    Depending on whether dev or prod chosen, different ip-address port is used.
    The server is run by default at `host 0.0.0.0` and `port 12892`. 
    This can be changed as a parameter in the `app.run` function which is called within that file. 
    By default the Server is started locally and in debug mode. To start the server in production mode, a production WSGI-Server should be used instead.

    `config` can be filled with values `dev` or `prod`. It is required to put the models for sequence-tagging and EDA as `best-model.pt` into `neuralNets/flair/resources/taggers/seq/` and `neuralNets/flair/resources/taggers/eda/` respectively.

2. Start frontend Web-Server, which accepts requests submitted 
    in the web form and forward it to the Flair-Server for processing:

    `python3 exmaples/web_client_external_partner_feedback.py`
    
    The frontend in `web_client_external_partner_feedback.py`  starts a web server with Flask, to send a request with the description of the case to the server to get a medical diagnose.
    The frontend contains a text box for the description and a button to submit it to the server.
    After encoding the request as `json` in the function `encode_request_json(text)`, the function `send_response(data)` is called where the request is sent to the flair-server for processing the request. 
    In `send_response(data)` the flair-server is specified by it its host-address, port and URL to be called. 

3. Call the ip and port of the started Web-Server from browser:
    
    `http://0.0.0.0:5000/predict`
    
    By calling the URL `http://0.0.0.0/predict` in a Web-Browser after starting this server by executing `web_client_external_partner_feedback.py`, the frontend for requesting the diagnose is shown. 
    If that URL is called, the server executes the function `api_feedback_with_model()` and it renders the frontend by utilizing `main.html`. 
    After submitting the request, the text is encoded as `json` before it is sent in the function `encode_request_json`. 
    The function `api_feedback_with_model()` is called again when the request is submitted, but with the text from the text box as parameter. 

    The results are sent back to `api_feedback_with_model()` where it shows the result under the form. The results contain the information for `content` and `reasoning`.  

    `main.html` which is used by this server to render the frontend, outputs the form with the text box and submit button. 
    When the flair-server sends a response, it extracts the information out of the response `json`-object and adds it to the HTML-`div`-tags with IDs `content` and `reasoning` the respective information.
    
4. Type in the text to be tagged and after submitting, the response is returned

# EDA Sequence-Tagging Training with flair

Steps:

1. Rename training files for training, testing and dev to `eda_train.txt`, `eda_test.txt` and `eda_dev.txt` and put them into folder `neuralNets/flair/data/`.
    
2. Start Training:

    `python3 neuralNets/flair/train_eda.py`
    
    In this process the following procedure is done:
     
    1. retrieve corpus
    2. specifying the tags to predict
    3. initialize word-, char- and flair-embeddings
    4. initialize sequence-tagger with embeddings, tag-types, tag-dictionary
    5. initialize model-trainer with sequence-tagger and corpus
    6. start training. Default file path for model is `resources/taggers/famulus_eda_test_n_bert_long2`
    7. plot loss and weights
