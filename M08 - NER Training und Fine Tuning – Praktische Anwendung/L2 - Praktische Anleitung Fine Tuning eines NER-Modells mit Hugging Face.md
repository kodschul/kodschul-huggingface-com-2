## Praktische Anleitung: Fine-Tuning eines NER-Modells mit Hugging Face

Named Entity Recognition (NER) ist eine zentrale Aufgabe im Bereich der natürlichen Sprachverarbeitung (NLP), bei der Entitäten wie Namen, Orte, Organisationen usw. in Texten erkannt und klassifiziert werden. Hugging Face bietet eine benutzerfreundliche Plattform, um vortrainierte Modelle für NER weiter anzupassen (Fine-Tuning), um sie für spezifische Anwendungsfälle zu optimieren.

Diese Anleitung führt Schritt für Schritt durch den Prozess des Fine-Tunings eines NER-Modells mit Hugging Face.

### 1. Vorbereitung der Umgebung

Zuerst müssen die notwendigen Pakete installiert werden. Dies umfasst die `transformers`-Bibliothek von Hugging Face und weitere Hilfsbibliotheken wie `datasets` und `torch`.

```bash
pip install transformers datasets torch
```

### 2. Auswahl und Laden eines vortrainierten Modells
Hugging Face bietet eine Vielzahl vortrainierter Modelle. Für NER ist das Modell bert-base-cased eine häufige Wahl, da es gut für Aufgaben der Sprachverarbeitung geeignet ist.

```python
from transformers import AutoTokenizer, AutoModelForTokenClassification

model_name = "bert-base-cased"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForTokenClassification.from_pretrained(model_name, num_labels=9)  # z.B. 9 Labels für NER

```

### 3. Vorbereitung des Datensatzes
Der nächste Schritt besteht darin, den Datensatz zu laden und für das Training vorzubereiten. Hugging Face bietet vorgefertigte Datensätze über die datasets-Bibliothek an, aber Sie können auch eigene Daten verwenden.


```python
from datasets import load_dataset

dataset = load_dataset("conll2003")

```

Der Datensatz muss tokenisiert und in das Format gebracht werden, das das Modell erwartet.

```python
def tokenize_and_align_labels(examples):
    tokenized_inputs = tokenizer(examples["tokens"], truncation=True, is_split_into_words=True)
    labels = []
    for i, label in enumerate(examples[f"ner_tags"]):
        word_ids = tokenized_inputs.word_ids(batch_index=i)
        previous_word_idx = None
        label_ids = []
        for word_idx in word_ids:
            if word_idx is None:
                label_ids.append(-100)
            elif word_idx != previous_word_idx:
                label_ids.append(label[word_idx])
            else:
                label_ids.append(-100)
            previous_word_idx = word_idx
        labels.append(label_ids)
    tokenized_inputs["labels"] = labels
    return tokenized_inputs

tokenized_datasets = dataset.map(tokenize_and_align_labels, batched=True)

```

### 4. Fine-Tuning des Modells
Nachdem der Datensatz vorbereitet wurde, können wir das Modell mit den spezifischen NER-Labels fine-tunen. Hugging Face stellt ein Trainer-API bereit, das diesen Prozess vereinfacht.

```python
from transformers import Trainer, TrainingArguments

training_args = TrainingArguments(
    output_dir="./results",
    evaluation_strategy="epoch",
    learning_rate=2e-5,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=16,
    num_train_epochs=3,
    weight_decay=0.01,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["validation"],
    tokenizer=tokenizer,
)

trainer.train()

```

### 5. Evaluierung des Modells
Nach dem Training kann das Modell mit dem Testdatensatz evaluiert werden, um die Leistung zu beurteilen. Dies umfasst typischerweise Metriken wie Präzision, Recall und F1-Score.

```python
results = trainer.evaluate()
print(f"Evaluation results: {results}")

```

### 6. Einsatz des Modells
Das feinabgestimmte Modell kann nun verwendet werden, um neue Texte zu analysieren und benannte Entitäten zu erkennen.

```python
text = "Hugging Face Inc. is a company based in New York City."
inputs = tokenizer(text, return_tensors="pt")
outputs = model(**inputs).logits
predictions = torch.argmax(outputs, dim=2)

predicted_tokens = [tokenizer.decode(ids) for ids in predictions[0].tolist()]
print(predicted_tokens)

```