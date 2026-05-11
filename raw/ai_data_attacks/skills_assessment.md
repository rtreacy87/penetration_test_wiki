# Skills Assessment

---

Your objective in this assessment is to strategically manipulate a model's training data to achieve a specific, nuanced misclassification behavior.

You will be provided a dataset in the zip attached to the question, along with a template notebook.

You will target a 4-class`One-vs-Rest`(`OvR`)`Logistic Regression`classifier. The goal is to poison its training dataset using only label flipping such that the subsequently trained model exhibits ambiguous classification for instances of`Class 1`. In other words, when the poisoned model encounters new data points that genuinely belong to`Class 1`, it should frequently misclassify them as either`Class 0`or`Class 2`, degrading the classification accuracy of`Class 1`.

Implement your attack strategy within the provided template notebook and use the same notebook to submit your poisoned model to the instancea api for evaluation.
