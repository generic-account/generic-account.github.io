# Amazon Textract Workshop for UCSF

## 1. Introduction to Amazon Textract

### What is Amazon Textract?

Amazon Textract is a fully managed machine learning service that automatically extracts text, handwriting, and structured data (tables, forms) from scanned documents. It goes beyond simple optical character recognition (OCR) by also identifying the context in which the data is presented, such as the relationship between fields in forms or the headers in tables.

### Why Textract is Useful for Researchers

Researchers at UCSF studying the JUUL Labs collection can benefit from Textract by automating the labor-intensive task of manually transcribing and organizing text from a vast array of documents. With Textract, you can quickly and accurately extract relevant data, allowing more time for analysis and interpretation.

### Overview of Workshop Goals

In this workshop, we will:
- Set up the environment to use Amazon Textract via Python.
- Extract text and structured data from documents.
- Clean and analyze the extracted data.
- Integrate Textract with Amazon Comprehend for further text analysis.
- Provide practical examples relevant to the JUUL Labs collection.

## 2. Setting Up the Environment

### Prerequisites

Before starting, ensure you have:
- An AWS account with access to Textract and Comprehend
- Basic familiarity with Python programming
- A SageMaker JupyterLabs environment

### Installing Necessary Python Libraries

We'll be using the `boto3` library to interact with AWS services, `Pillow` for image handling, and the AWS Textract libraries. The code below is how you would install this with iPython in a Jupyter notebook, but you could also pip install these libraries through your terminal.

```python
!pip install boto3
!pip install pillow
!python -m pip install -q amazon-textract-response-parser --upgrade
!python -m pip install -q amazon-textract-caller --upgrade
!python -m pip install -q amazon-textract-prettyprinter==0.0.16
!python -m pip install -q amazon-textract-textractor --upgrade
```

Now, if you are using iPython, restart the kernel to ensure we can access these libraries.

```python
import IPython
IPython.Application.instance().kernel.do_shutdown(True)
```

### Configuring AWS

To interact with AWS, you need to set up your AWS credentials, which we'll do through SageMaker. First, import all the necessary libraries

```python
import boto3
import botocore
import sagemaker
import pandas as pd
from IPython.display import Image, display, JSON
from textractcaller.t_call import call_textract, Textract_Features, call_textract_expense
from textractprettyprinter.t_pretty_print import convert_table_to_list
from trp import Document
import os
```

Then, set the necessary variables and get a Textract and Comprehend client. This retrieves information from the SageMaker session, so this exact code won't work if you're not on SageMaker. If you're not, you would need to set up your credentials through the AWS CLI.

```python
data_bucket = sagemaker.Session().default_bucket()
region = boto3.session.Session().region_name
account_id = boto3.client('sts').get_caller_identity().get('Account')

os.environ["BUCKET"] = data_bucket
os.environ["REGION"] = region
role = sagemaker.get_execution_role()

s3 = boto3.client('s3')
textract_client = boto3.client('textract', region_name=region)
comprehend_client = boto3.client('comprehend', region_name=region)
```

## 3. Exploring Amazon Textract Capabilities

### Text Extraction from Documents

Textract can extract raw text from various types of documents. This includes printed text as well as handwritten notes.

#### Example: Extracting Text from a Sample Document

```python
def extract_text_from_document(image_path):
    with open(image_path, 'rb') as document:
        response = textract_client.detect_document_text(Document={'Bytes': document.read()})
    
    extracted_text = ''
    for item in response['Blocks']:
        if item['BlockType'] == 'LINE':
            extracted_text += item['Text'] + '\n'
    
    return extracted_text

# Example usage
image_path = 'path/to/your/document-image.png'
extracted_text = extract_text_from_document(image_path)
print(extracted_text)
```

### Extracting Structured Data: Tables and Forms

Textract can also detect and extract structured data from tables and forms within documents.

#### Example: Extracting Tables

```python
def extract_tables_from_document(image_path):
    with open(image_path, 'rb') as document:
        response = textract_client.analyze_document(Document={'Bytes': document.read()}, FeatureTypes=['TABLES'])
    
    tables = []
    for block in response['Blocks']:
        if block['BlockType'] == 'TABLE':
            table = []
            for relationship in block['Relationships']:
                for id in relationship['Ids']:
                    cell = next(item for item in response['Blocks'] if item['Id'] == id)
                    table.append(cell['Text'])
            tables.append(table)
    
    return tables

# Example usage
tables = extract_tables_from_document(image_path)
print(tables)
```

### Working with PDFs

Often, we won't have an image file we can directly work with. Especially with IDL documents, we'll often encounter PDFs that we can't easily convert with Textract. So, we'll use this helper function below to convert a PDF to a series of PNGs. Keep in mind, this stores all the PNGs which may use more storage. We could also use a bytearray, but that introduces excessive complexity.

This function converts one PDF to a series of PNGs and puts them in output_folder. DPI is an integer specifying the resolution of the images. A lower resolution will save space, but too low of a resolution will decrease Textract performance. The function returns a list of image paths.

```python
from pdf2image import convert_from_path

def convert_pdf_to_pngs(pdf_path, output_folder='output_images', dpi=300):
    images = convert_from_path(pdf_path, dpi=dpi)
    image_paths = []
    
    for i, image in enumerate(images):
        image_path = f'{output_folder}/page_{i+1}.png'
        image.save(image_path, 'PNG')
        image_paths.append(image_path)
    
    return image_paths
```

Now, we'll use a helper function that extracts OCR from a list of PNGs.

```python
def extract_text_from_png_list(image_paths):
    extracted_text = ''
    
    for image_path in image_paths:
        text = extract_text_from_document(image_path)
        extracted_text += text + '\n'
    
    return extracted_text
```

## 4. Post-Processing and Analyzing Extracted Data

### Cleaning and Normalizing Text Data

Once the text is extracted, it often needs cleaning and normalization to be useful for analysis.

```python
def clean_extracted_text(text):
    # Example cleaning process: remove excessive whitespace and normalize case
    cleaned_text = text.replace('\n', ' ').strip().lower()
    return cleaned_text

# Example usage
cleaned_text = clean_extracted_text(extracted_text)
print(cleaned_text)
```

### Keyword Analysis

For researchers focusing on specific themes, such as the impact of nicotine or vaping, it can be useful to analyze the frequency and context of certain keywords.

```python
def analyze_text_for_keywords(text, keywords):
    results = {}
    for keyword in keywords:
        occurrences = text.count(keyword)
        results[keyword] = occurrences
    return results

# Example usage
keywords = ['nicotine', 'vaping', 'health']
analysis_results = analyze_text_for_keywords(cleaned_text, keywords)
print(analysis_results)
```

### Sentiment Analysis with Amazon Comprehend

Amazon Comprehend can be integrated with Textract to perform sentiment analysis on the extracted text. This can be useful for identifying the bias of documents.

```python
def analyze_sentiment(text):
    response = comprehend_client.detect_sentiment(Text=text, LanguageCode='en')
    return response['Sentiment']

# Example usage
sentiment = analyze_sentiment(cleaned_text)
print(f"Sentiment of the document: {sentiment}")
```

## 5. Advanced Use Cases

### Combining Textract and Comprehend for Entity Recognition

Beyond sentiment, Comprehend can also recognize entities such as organizations, locations, and people mentioned in the text.

```python
def analyze_entities(text):
    response = comprehend_client.detect_entities(Text=text, LanguageCode='en')
    entities = [(entity['Text'], entity['Type']) for entity in response['Entities']]
    return entities

# Example usage
entities = analyze_entities(cleaned_text)
print(f"Entities found: {entities}")
```

### Query-based Extraction

Honestly, this is one of my favorite features. We can extract data according to natural language queries we pass in. This can be perfect for easy automated extraction of simple information, but also extraction of more nuanced data. This function provides a utility for extraction based on queries based on a document path, and a list of queries.

```python
from textractcaller import QueriesConfig, Query, call_textract
import trp.trp2 as t2

def query_based_extraction(document_path, queries):
    # Setup the queries
    textract_queries = [Query(text=query['text'], alias=query['alias'], pages=query['pages']) for query in queries]
    
    # Setup the query config with the above queries
    queries_config = QueriesConfig(queries=textract_queries)
    
    # Read the document
    with open(document_path, 'rb') as document:
        imageBytes = bytearray(document.read())
    
    # Call Textract with the queries
    response = call_textract(input_document=imageBytes, features=[t2.Textract_Features.QUERIES], queries_config=queries_config)
    doc_ev = t2.TDocumentSchema().load(response)
    
    # Extract and return query answers
    entities = {}
    for page in doc_ev.pages:
        query_answers = doc_ev.get_query_answers(page=page)
        if query_answers:
            for answer in query_answers:
                entities[answer[1]] = answer[2]
    
    return entities
```

Here's an example usage. The text is the natural language query, and the alias is simply a name we give to the result of the query.

```python
# Define the queries
queries = [
    {"text": "Who is the applicant's date of employment?", "alias": "EMPLOYMENT_DATE", "pages": ["1"]},
    {"text": "What is the probability of continued employment?", "alias": "CONTINUED_EMPLOYMENT_PROB", "pages": ["1"]}
]

# Path to your document
document_path = "./path/to/employment_document.png"

# Perform the query-based extraction
extracted_entities = query_based_extraction(document_path, queries)

# Display the results
print(extracted_entities)
```

### Practical Example: Analyzing JUUL Labs Documents

Let's combine everything we've learned so far to analyze a document from the JUUL Labs collection.

```python
# Load and extract text
image_path = 'path/to/juul/document.png'
extracted_text = extract_text_from_document(image_path)

# Clean the text
cleaned_text = clean_extracted_text(extracted_text)

# Analyze keywords
keywords = ['nicotine', 'vaping', 'health']
keyword_analysis = analyze_text_for_keywords(cleaned_text, keywords)

# Analyze sentiment
sentiment = analyze_sentiment(cleaned_text)

# Analyze entities
entities = analyze_entities(cleaned_text)

# Display results
print("Extracted Text:\n", cleaned_text)
print("Keyword Analysis Results:\n", keyword_analysis)
print("Document Sentiment:\n", sentiment)
print("Entities Detected:\n", entities)
```

### Handling Large Document Collections

For large collections, such as the JUUL Labs dataset, you can batch process documents and store results for further analysis.

```python
def process_documents_in_directory(directory_path):
    results = []
    for filename in os.listdir(directory_path):
        if filename.endswith(('.png', '.jpg', '.jpeg')):
            image_path = os.path.join(directory_path, filename)
            text = extract_text_from_document(image_path)
            cleaned_text = clean_extracted_text(text)
            sentiment = analyze_sentiment(cleaned_text)
            keyword_analysis = analyze_text_for_keywords(cleaned_text, keywords)
            entities = analyze_entities(cleaned_text)
            results.append({
                'filename': filename,
                'sentiment': sentiment,
                'keywords': keyword_analysis,
                'entities': entities
            })
    return results

# Example usage
directory_path = 'path/to/juul/collection'
document_analysis_results = process_documents_in_directory(directory_path)
for result in document_analysis_results:
    print(result)
```

## 6. Further Resources and Next Steps

This only scratched the surface of the power of AWS tools like Textract and Comprehend. They can also perform a variety of tasks from key-value pair extraction (e.g. getting somebody's birth date from a form), signature identification, and even PHI and PII identification with Comprehend Medical (which would be of particular relevance to UCSF).

- **Amazon Textract Documentation**: [Learn more about Textract capabilities](https://aws.amazon.com/textract/).
- **Amazon Comprehend Documentation**: [Explore text analysis with Comprehend](https://aws.amazon.com/comprehend/).
- **Next Steps**: Consider automating the process with AWS Lambda for scalable workflows or integrating with Amazon S3 for document storage.

## 7. Conclusion

This workshop provided a brief introduction to using Amazon Textract and Comprehend for document processing and analysis. These are powerful tools in streamlining and enhancing research workflows, especially when dealing with large document collections like those the IDL has.

By automating text extraction, cleaning, and basic analysis, important documents can quickly be identified and more time can be devoted to drawing meaningful insights from the data.
