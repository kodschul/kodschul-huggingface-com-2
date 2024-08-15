## Vorbereitung von Datensätzen für das NER-Training

Named Entity Recognition (NER) ist eine zentrale Aufgabe im Bereich der Verarbeitung natürlicher Sprache (Natural Language Processing, NLP), bei der es darum geht, benannte Entitäten wie Personen, Organisationen, Orte usw. in einem Text zu erkennen und zu klassifizieren. Ein entscheidender Schritt für das Training eines NER-Modells ist die sorgfältige Vorbereitung der Datensätze. In dieser Schulung werden die wesentlichen Schritte zur Vorbereitung von Datensätzen für das NER-Training unter Verwendung der HuggingFace-Bibliothek beschrieben.

### 1. Datensammlung

Der erste Schritt besteht darin, geeignete Textdaten zu sammeln, die benannte Entitäten enthalten. Diese Texte können aus verschiedenen Quellen stammen, wie Nachrichtenartikeln, wissenschaftlichen Papieren, sozialen Medien usw. Es ist wichtig, eine breite und repräsentative Auswahl an Texten zu haben, um ein robustes NER-Modell zu trainieren.

- **Beispiel**: Texte aus verschiedenen Nachrichtenquellen, die Personen, Organisationen und Orte erwähnen.

### 2. Datenannotation

Nach der Sammlung der Texte müssen diese annotiert werden. Dies bedeutet, dass die benannten Entitäten in den Texten manuell markiert und den entsprechenden Kategorien zugeordnet werden. Diese Annotationen sind notwendig, um das Modell darauf zu trainieren, ähnliche Entitäten in neuen Texten zu erkennen.

- **Beispiel**: In einem Satz wie „Angela Merkel besuchte Berlin“ werden „Angela Merkel“ als PERSON und „Berlin“ als LOCATION annotiert.

#### Tools für die Annotation

Es gibt mehrere Tools und Plattformen, die die Annotation von NER-Daten erleichtern:

- **BRAT**: Ein einfaches webbasiertes Tool zur Textannotation.
- **Prodigy**: Ein kommerzielles Tool, das maschinelles Lernen integriert, um die Annotation zu beschleunigen.
- **Label Studio**: Ein Open-Source-Tool zur flexiblen Datenannotation.

### 3. Datenformatierung

Nach der Annotation müssen die Daten in ein Format gebracht werden, das für das Training eines NER-Modells geeignet ist. Ein gängiges Format ist das **IOB-Format (Inside-Outside-Beginning)**, bei dem jeder Token eines Satzes mit einer speziellen Markierung versehen wird, die angibt, ob es sich um den Anfang (B), das Innere (I) oder kein Teil (O) einer Entität handelt.

- **Beispiel**:

    ```plaintext
    Angela  B-PER
    Merkel  I-PER
    besuchte  O
    Berlin  B-LOC
    ```

In diesem Beispiel wird „Angela Merkel“ als PERSON (B-PER und I-PER) und „Berlin“ als LOCATION (B-LOC) markiert.

### 4. Laden und Vorverarbeiten der Daten mit HuggingFace

HuggingFace bietet leistungsfähige Tools, um annotierte Datensätze zu laden und für das NER-Training vorzubereiten.

#### a) Datensatz laden

Mit der `datasets`-Bibliothek von HuggingFace können Sie den annotierten Datensatz direkt laden und verarbeiten.

```python
from datasets import load_dataset

dataset = load_dataset('path/to/your/dataset')
```

#### b) Tokenisierung
Die Texte müssen tokenisiert werden, bevor sie dem Modell zugeführt werden. Hierbei wird der Text in einzelne Token (Wörter oder Subwörter) zerlegt.

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")
tokenized_dataset = dataset.map(lambda x: tokenizer(x['text']), batched=True)

```

#### c) Alignment von Labels
Beim Tokenisieren von Texten müssen die Labels den Token angepasst werden, besonders wenn Subtoken verwendet werden. Dies ist ein kritischer Schritt, um sicherzustellen, dass die Labels korrekt den Token zugeordnet bleiben.

```python
def align_labels_with_tokens(labels, word_ids):
    aligned_labels = []
    current_word = None
    for word_id in word_ids:
        if word_id != current_word:
            current_word = word_id
            aligned_labels.append(labels[word_id])
        else:
            aligned_labels.append(-100)
    return aligned_labels

tokenized_dataset = tokenized_dataset.map(
    lambda x: {'labels': align_labels_with_tokens(x['labels'], x['word_ids'])},
    batched=True
)

```

### 5. Datenaufteilung
Der annotierte Datensatz sollte in Trainings-, Validierungs- und Testdatensätze aufgeteilt werden, um das Modell zu trainieren und seine Leistung zu bewerten.

Beispiel: Eine übliche Aufteilung ist 70% Training, 15% Validierung, 15% Testen.

```python
train_testvalid = dataset.train_test_split(test_size=0.3)
valid_test = train_testvalid['test'].train_test_split(test_size=0.5)
dataset['train'] = train_testvalid['train']
dataset['valid'] = valid_test['train']
dataset['test'] = valid_test['test']

```