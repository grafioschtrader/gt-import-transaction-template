## Preface
For the main project, refer to [Grafioschtrader](https://github.com/grafioschtrader/grafioschtrader) where these import templates are used.

# Import transaction template
Most trading platforms provide an export function of transactions. The result of this export can possibly be imported into GT. Since each trading platform implements its own design of **documents**, an import template must be created in GT according to each of these documents. GT can handle two document types **PDF** and **CSV**.

## Import implementation
In most cases, the import function generally implemented in GT with the corresponding import templates is sufficient to process the documents of a trading platform. In all other cases, an import function must be implemented in GT.

## Why this repository
GT's goal with these **import templates** is to generalize the transaction import as much as possible. This avoids many specific implementations per trading platform. In addition, a document change of the trading platform leads only in a few cases to an update of GT, which is very useful for the user and software developer.

First, examples of CSV or PDF documents are collected, which are to be imported into GT. After that, ambitious users create appropriate import templates or, in rare cases, software developers program a special implementation for the platform. In the end, the import templates are published in this repository and/or GT receives an update in rare cases.

## Use of existing import templates
1. Check in this repository if your trading platform is listed and read the corresponding "README". It may be that according to the information in this "README" import templates are not yet available or the required specific implementation is not yet available.
2. If the import templates are available, they can be assigned to the appropriate trading platform in GT using drag and drop. More information can be found at "[Import Vorlagengruppe](https://grafioschtrader.github.io/gt-user-manual/de/basedata/imptranstemplate/)", currently unfortunately only in German.

## LLM-Assisted Template Creation
Creating PDF import templates manually can be tedious. The [LLM PDF Template Specification](LLM_PDF_TEMPLATE_SPEC.md) provides a comprehensive specification that enables a Large Language Model to **automatically generate valid GT import templates** when given PDF-to-text conversions as input. Copy its content into your LLM conversation as context, then provide your PDF-to-text documents as input.

## Send us your PDF or CSV documents with or without import templates
In order for us to improve GT regarding the transaction import, we need your help. You can send us PDF (transformed as text) or CSV documents anonymously with or without import template. Your documents should be pseudonymized, i.e. changed address data and modified account and deposit number. We recommend you to use [GUERRILLAMAIL](//www.guerrillamail.com/compose) email with the addressee grafiosch@gmail.com. We have chosen GUERRILLAMAIL because it allows attachments.

### Without import templates
Thank you for your anonymized documents. We expect the **transformed PDF documents** to be **complete** and **pseudonymized**. It is very advantageous when multiple documents are sent from different transactions. For transformation and pseudonymization of multiple PDF documents we recommend using the [GT-PDF-Transform](//github.com/grafioschtrader/gt-pdf-transform#gt-pdf-transform) program. The **subject** of the email should contain the **name of the trading platform**. You can send the documents as an attachment or as the text of the email. For more information, see "[Ihre Dokumente für das GT-Projekt](//grafioschtrader.github.io/gt-user-manual/de/tenantportfolio/securityaccounts/transactionimport/)" unfortunately only in German at this time.

#### Do not do the following
The conversion is done with Apache [PDFBox](//pdfbox.apache.org/) in the current version 2.0.24 with the option "**Sorted by position**". This means that the PDF documents converted to text at Portfolio Performance cannot be used by GT as a basis for creating import templates.

### Including import templates
Thank you for providing us with your created import templates. Please also provide the corresponding PDF or CSV file/s, where the **PDF document** should be **transformed** and **pseudonymized**. The unabridged and transformed PDF documents can be sent as email text with the heading of the file name of the corresponding import template. The **subject** should contain the word "**Template:**" and the **file name** of the exported zip file. You can send the whole zip file or a single import template with "tmpl" file extension. For more information, see "[Die Importvorlage veröffentlichen](//grafioschtrader.github.io/gt-user-manual/de/basedata/imptranstemplate/)" unfortunately only in German at this time.
