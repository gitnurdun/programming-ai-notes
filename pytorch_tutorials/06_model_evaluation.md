# Model Evaluation in PyTorch - 30+ Questions and Answers

## 1. What is the difference between validation and testing in model evaluation?

**Answer:** Validation is used during training to tune hyperparameters and prevent overfitting. Testing is performed after training to assess the final model's performance on unseen data. Validation data influences the training process (e.g., early stopping, hyperparameter tuning), while test data should only be used once for final evaluation to get an unbiased performance estimate.

## 2. How do you implement a validation loop in PyTorch?

**Answer:**
```python
def validate_model(model, val_loader, criterion, device):
    model.eval()  # Set to evaluation mode
    val_loss = 0.0
    correct = 0
    total = 0
    
    with torch.no_grad():  # Disable gradient computation
        for inputs, targets in val_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            val_loss += loss.item()
            
            _, predicted = torch.max(outputs.data, 1)
            total += targets.size(0)
            correct += (predicted == targets).sum().item()
    
    avg_loss = val_loss / len(val_loader)
    accuracy = 100 * correct / total
    
    return avg_loss, accuracy
```

## 3. How do you evaluate a model on test data properly?

**Answer:**
```python
def test_model(model, test_loader, criterion, device):
    model.eval()
    test_loss = 0.0
    correct = 0
    total = 0
    all_predictions = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in test_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            test_loss += loss.item()
            
            _, predicted = torch.max(outputs.data, 1)
            total += targets.size(0)
            correct += (predicted == targets).sum().item()
            
            # Store for detailed metrics
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    avg_loss = test_loss / len(test_loader)
    accuracy = 100 * correct / total
    
    return {
        'loss': avg_loss,
        'accuracy': accuracy,
        'predictions': all_predictions,
        'targets': all_targets
    }
```

## 4. What are the common evaluation metrics for classification tasks?

**Answer:** Common classification metrics include:
- **Accuracy:** (TP + TN) / (TP + TN + FP + FN)
- **Precision:** TP / (TP + FP)
- **Recall (Sensitivity):** TP / (TP + FN)
- **F1-Score:** 2 * (Precision * Recall) / (Precision + Recall)
- **Specificity:** TN / (TN + FP)
- **ROC-AUC:** Area under the ROC curve
- **Confusion Matrix:** Detailed breakdown of predictions

## 5. How do you calculate precision, recall, and F1-score in PyTorch?

**Answer:**
```python
from sklearn.metrics import precision_score, recall_score, f1_score, confusion_matrix

def calculate_classification_metrics(model, data_loader, device, num_classes=2):
    model.eval()
    all_predictions = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            _, predicted = torch.max(outputs, 1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    # Calculate metrics
    precision = precision_score(all_targets, all_predictions, average='weighted')
    recall = recall_score(all_targets, all_predictions, average='weighted')
    f1 = f1_score(all_targets, all_predictions, average='weighted')
    cm = confusion_matrix(all_targets, all_predictions)
    
    return {
        'precision': precision,
        'recall': recall,
        'f1_score': f1,
        'confusion_matrix': cm
    }
```

## 6. How do you implement ROC-AUC calculation for binary classification?

**Answer:**
```python
from sklearn.metrics import roc_auc_score, roc_curve
import matplotlib.pyplot as plt

def calculate_roc_auc(model, data_loader, device):
    model.eval()
    all_probabilities = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            
            # Get probabilities for positive class
            probabilities = torch.softmax(outputs, dim=1)[:, 1]
            
            all_probabilities.extend(probabilities.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    # Calculate ROC-AUC
    auc_score = roc_auc_score(all_targets, all_probabilities)
    
    # Calculate ROC curve points
    fpr, tpr, thresholds = roc_curve(all_targets, all_probabilities)
    
    return {
        'auc_score': auc_score,
        'fpr': fpr,
        'tpr': tpr,
        'thresholds': thresholds
    }

def plot_roc_curve(fpr, tpr, auc_score):
    plt.figure(figsize=(8, 6))
    plt.plot(fpr, tpr, color='darkorange', lw=2, 
             label=f'ROC curve (AUC = {auc_score:.2f})')
    plt.plot([0, 1], [0, 1], color='navy', lw=2, linestyle='--')
    plt.xlim([0.0, 1.0])
    plt.ylim([0.0, 1.05])
    plt.xlabel('False Positive Rate')
    plt.ylabel('True Positive Rate')
    plt.title('Receiver Operating Characteristic')
    plt.legend(loc="lower right")
    plt.show()
```

## 7. How do you evaluate multi-class classification models?

**Answer:**
```python
from sklearn.metrics import classification_report, precision_recall_fscore_support

def evaluate_multiclass_model(model, data_loader, device, class_names=None):
    model.eval()
    all_predictions = []
    all_targets = []
    all_probabilities = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            
            # Get predictions
            _, predicted = torch.max(outputs, 1)
            
            # Get probabilities
            probabilities = torch.softmax(outputs, dim=1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
            all_probabilities.extend(probabilities.cpu().numpy())
    
    # Detailed classification report
    report = classification_report(
        all_targets, all_predictions, 
        target_names=class_names, 
        output_dict=True
    )
    
    # Per-class metrics
    precision, recall, f1, support = precision_recall_fscore_support(
        all_targets, all_predictions, average=None
    )
    
    return {
        'classification_report': report,
        'precision': precision,
        'recall': recall,
        'f1_score': f1,
        'support': support,
        'predictions': all_predictions,
        'targets': all_targets,
        'probabilities': all_probabilities
    }
```

## 8. How do you calculate mean squared error and R-squared for regression models?

**Answer:**
```python
from sklearn.metrics import mean_squared_error, r2_score

def evaluate_regression_model(model, data_loader, device):
    model.eval()
    all_predictions = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            
            all_predictions.extend(outputs.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    # Calculate metrics
    mse = mean_squared_error(all_targets, all_predictions)
    rmse = np.sqrt(mse)
    r2 = r2_score(all_targets, all_predictions)
    mae = np.mean(np.abs(np.array(all_targets) - np.array(all_predictions)))
    
    return {
        'mse': mse,
        'rmse': rmse,
        'r2_score': r2,
        'mae': mae,
        'predictions': all_predictions,
        'targets': all_targets
    }
```

## 9. How do you implement cross-validation for model evaluation?

**Answer:**
```python
from sklearn.model_selection import KFold

def cross_validate_model(model_class, dataset, criterion, optimizer_class, 
                        cv_folds=5, epochs=10, batch_size=32):
    kfold = KFold(n_splits=cv_folds, shuffle=True, random_state=42)
    cv_scores = []
    
    for fold, (train_idx, val_idx) in enumerate(kfold.split(dataset)):
        print(f"Fold {fold + 1}/{cv_folds}")
        
        # Create data loaders for this fold
        train_subset = torch.utils.data.Subset(dataset, train_idx)
        val_subset = torch.utils.data.Subset(dataset, val_idx)
        
        train_loader = DataLoader(train_subset, batch_size=batch_size, shuffle=True)
        val_loader = DataLoader(val_subset, batch_size=batch_size, shuffle=False)
        
        # Create new model instance for this fold
        model = model_class()
        optimizer = optimizer_class(model.parameters())
        
        # Train model
        for epoch in range(epochs):
            train_model(model, train_loader, optimizer, criterion, device)
        
        # Evaluate model
        val_metrics = validate_model(model, val_loader, criterion, device)
        cv_scores.append(val_metrics['accuracy'])
        
        print(f"Fold {fold + 1} Accuracy: {val_metrics['accuracy']:.2f}%")
    
    mean_score = np.mean(cv_scores)
    std_score = np.std(cv_scores)
    
    return {
        'cv_scores': cv_scores,
        'mean_accuracy': mean_score,
        'std_accuracy': std_score
    }
```

## 10. How do you implement a confusion matrix visualization?

**Answer:**
```python
import seaborn as sns
import matplotlib.pyplot as plt
from sklearn.metrics import confusion_matrix

def plot_confusion_matrix(y_true, y_pred, class_names=None, normalize=False):
    cm = confusion_matrix(y_true, y_pred)
    
    if normalize:
        cm = cm.astype('float') / cm.sum(axis=1)[:, np.newaxis]
        fmt = '.2f'
    else:
        fmt = 'd'
    
    plt.figure(figsize=(10, 8))
    sns.heatmap(cm, annot=True, fmt=fmt, cmap='Blues',
                xticklabels=class_names, yticklabels=class_names)
    plt.title('Confusion Matrix')
    plt.xlabel('Predicted Label')
    plt.ylabel('True Label')
    plt.show()
    
    return cm

def evaluate_with_confusion_matrix(model, data_loader, device, class_names=None):
    model.eval()
    all_predictions = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            _, predicted = torch.max(outputs, 1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    # Plot confusion matrix
    cm = plot_confusion_matrix(all_targets, all_predictions, class_names)
    
    return {
        'confusion_matrix': cm,
        'predictions': all_predictions,
        'targets': all_targets
    }
```

## 11. How do you calculate precision-recall curves?

**Answer:**
```python
from sklearn.metrics import precision_recall_curve, average_precision_score

def calculate_precision_recall_curve(model, data_loader, device):
    model.eval()
    all_probabilities = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            
            # Get probabilities for positive class (binary classification)
            probabilities = torch.softmax(outputs, dim=1)[:, 1]
            
            all_probabilities.extend(probabilities.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    # Calculate precision-recall curve
    precision, recall, thresholds = precision_recall_curve(all_targets, all_probabilities)
    
    # Calculate average precision
    avg_precision = average_precision_score(all_targets, all_probabilities)
    
    return {
        'precision': precision,
        'recall': recall,
        'thresholds': thresholds,
        'average_precision': avg_precision
    }

def plot_precision_recall_curve(precision, recall, avg_precision):
    plt.figure(figsize=(8, 6))
    plt.plot(recall, precision, color='b', lw=2,
             label=f'Precision-Recall curve (AP = {avg_precision:.2f})')
    plt.xlabel('Recall')
    plt.ylabel('Precision')
    plt.title('Precision-Recall Curve')
    plt.legend(loc="lower left")
    plt.grid(True)
    plt.show()
```

## 12. How do you evaluate models with imbalanced datasets?

**Answer:**
```python
from sklearn.metrics import classification_report, precision_recall_fscore_support

def evaluate_imbalanced_model(model, data_loader, device, class_names=None):
    model.eval()
    all_predictions = []
    all_targets = []
    all_probabilities = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            probabilities = torch.softmax(outputs, dim=1)
            _, predicted = torch.max(outputs, 1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
            all_probabilities.extend(probabilities.cpu().numpy())
    
    # Calculate metrics for each class
    precision, recall, f1, support = precision_recall_fscore_support(
        all_targets, all_predictions, average=None
    )
    
    # Macro and weighted averages
    macro_precision = precision_score(all_targets, all_predictions, average='macro')
    macro_recall = recall_score(all_targets, all_predictions, average='macro')
    macro_f1 = f1_score(all_targets, all_predictions, average='macro')
    
    weighted_precision = precision_score(all_targets, all_predictions, average='weighted')
    weighted_recall = recall_score(all_targets, all_predictions, average='weighted')
    weighted_f1 = f1_score(all_targets, all_predictions, average='weighted')
    
    # Per-class accuracy
    class_accuracy = []
    for i in range(len(np.unique(all_targets))):
        class_mask = np.array(all_targets) == i
        if np.sum(class_mask) > 0:
            class_acc = np.mean(np.array(all_predictions)[class_mask] == i)
            class_accuracy.append(class_acc)
        else:
            class_accuracy.append(0.0)
    
    return {
        'per_class_precision': precision,
        'per_class_recall': recall,
        'per_class_f1': f1,
        'per_class_support': support,
        'per_class_accuracy': class_accuracy,
        'macro_precision': macro_precision,
        'macro_recall': macro_recall,
        'macro_f1': macro_f1,
        'weighted_precision': weighted_precision,
        'weighted_recall': weighted_recall,
        'weighted_f1': weighted_f1,
        'classification_report': classification_report(
            all_targets, all_predictions, target_names=class_names
        )
    }
```

## 13. How do you implement bootstrapping for confidence intervals?

**Answer:**
```python
def bootstrap_evaluation(model, dataset, criterion, device, n_bootstrap=1000, sample_size=None):
    if sample_size is None:
        sample_size = len(dataset)
    
    bootstrap_scores = []
    
    for i in range(n_bootstrap):
        # Sample with replacement
        indices = np.random.choice(len(dataset), size=sample_size, replace=True)
        bootstrap_dataset = torch.utils.data.Subset(dataset, indices)
        bootstrap_loader = DataLoader(bootstrap_dataset, batch_size=32, shuffle=False)
        
        # Evaluate on bootstrap sample
        metrics = validate_model(model, bootstrap_loader, criterion, device)
        bootstrap_scores.append(metrics['accuracy'])
    
    # Calculate confidence intervals
    confidence_interval = np.percentile(bootstrap_scores, [2.5, 97.5])
    mean_score = np.mean(bootstrap_scores)
    std_score = np.std(bootstrap_scores)
    
    return {
        'bootstrap_scores': bootstrap_scores,
        'mean_accuracy': mean_score,
        'std_accuracy': std_score,
        'confidence_interval': confidence_interval,
        'lower_bound': confidence_interval[0],
        'upper_bound': confidence_interval[1]
    }
```

## 14. How do you evaluate models on multiple metrics simultaneously?

**Answer:**
```python
def comprehensive_evaluation(model, data_loader, device, task_type='classification'):
    model.eval()
    
    if task_type == 'classification':
        return evaluate_classification_comprehensive(model, data_loader, device)
    elif task_type == 'regression':
        return evaluate_regression_comprehensive(model, data_loader, device)
    else:
        raise ValueError("task_type must be 'classification' or 'regression'")

def evaluate_classification_comprehensive(model, data_loader, device):
    all_predictions = []
    all_targets = []
    all_probabilities = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            probabilities = torch.softmax(outputs, dim=1)
            _, predicted = torch.max(outputs, 1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
            all_probabilities.extend(probabilities.cpu().numpy())
    
    # Calculate all metrics
    accuracy = accuracy_score(all_targets, all_predictions)
    precision = precision_score(all_targets, all_predictions, average='weighted')
    recall = recall_score(all_targets, all_predictions, average='weighted')
    f1 = f1_score(all_targets, all_predictions, average='weighted')
    
    # For binary classification, calculate AUC
    if len(np.unique(all_targets)) == 2:
        auc = roc_auc_score(all_targets, np.array(all_probabilities)[:, 1])
    else:
        # For multi-class, use one-vs-rest
        auc = roc_auc_score(all_targets, all_probabilities, multi_class='ovr')
    
    # Confusion matrix
    cm = confusion_matrix(all_targets, all_predictions)
    
    return {
        'accuracy': accuracy,
        'precision': precision,
        'recall': recall,
        'f1_score': f1,
        'auc': auc,
        'confusion_matrix': cm,
        'predictions': all_predictions,
        'targets': all_targets,
        'probabilities': all_probabilities
    }
```

## 15. How do you implement model evaluation with detailed logging?

**Answer:**
```python
import logging
import json
from datetime import datetime

def setup_evaluation_logger(log_file='evaluation.log'):
    logging.basicConfig(
        filename=log_file,
        level=logging.INFO,
        format='%(asctime)s - %(levelname)s - %(message)s'
    )
    return logging.getLogger(__name__)

def evaluate_and_log(model, data_loader, device, experiment_name, log_dir='./logs'):
    logger = setup_evaluation_logger(f"{log_dir}/{experiment_name}_eval.log")
    
    # Start evaluation
    start_time = datetime.now()
    logger.info(f"Starting evaluation for {experiment_name}")
    logger.info(f"Model: {model.__class__.__name__}")
    logger.info(f"Dataset size: {len(data_loader.dataset)}")
    logger.info(f"Number of batches: {len(data_loader)}")
    
    # Perform evaluation
    results = comprehensive_evaluation(model, data_loader, device)
    
    # Log results
    logger.info(f"Evaluation completed in {datetime.now() - start_time}")
    logger.info(f"Accuracy: {results['accuracy']:.4f}")
    logger.info(f"Precision: {results['precision']:.4f}")
    logger.info(f"Recall: {results['recall']:.4f}")
    logger.info(f"F1-Score: {results['f1_score']:.4f}")
    logger.info(f"AUC: {results['auc']:.4f}")
    
    # Save detailed results
    results_file = f"{log_dir}/{experiment_name}_results.json"
    results_to_save = {
        'experiment_name': experiment_name,
        'timestamp': start_time.isoformat(),
        'metrics': {
            'accuracy': float(results['accuracy']),
            'precision': float(results['precision']),
            'recall': float(results['recall']),
            'f1_score': float(results['f1_score']),
            'auc': float(results['auc'])
        },
        'model_info': {
            'model_class': model.__class__.__name__,
            'num_parameters': sum(p.numel() for p in model.parameters())
        }
    }
    
    with open(results_file, 'w') as f:
        json.dump(results_to_save, f, indent=2)
    
    logger.info(f"Results saved to {results_file}")
    
    return results
```

## 16. How do you compare multiple models during evaluation?

**Answer:**
```python
def compare_models(models_dict, data_loader, device, model_names=None):
    if model_names is None:
        model_names = list(models_dict.keys())
    
    comparison_results = {}
    
    for name, model in models_dict.items():
        print(f"Evaluating model: {name}")
        results = comprehensive_evaluation(model, data_loader, device)
        comparison_results[name] = results
        print(f"  Accuracy: {results['accuracy']:.4f}")
        print(f"  F1-Score: {results['f1_score']:.4f}")
        print(f"  AUC: {results['auc']:.4f}")
        print()
    
    return comparison_results

def plot_model_comparison(comparison_results, metric='accuracy'):
    model_names = list(comparison_results.keys())
    values = [comparison_results[name][metric] for name in model_names]
    
    plt.figure(figsize=(10, 6))
    bars = plt.bar(model_names, values)
    plt.ylabel(metric.capitalize())
    plt.title(f'Model Comparison - {metric.capitalize()}')
    plt.xticks(rotation=45)
    
    # Add value labels on bars
    for bar, value in zip(bars, values):
        plt.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.01,
                f'{value:.3f}', ha='center', va='bottom')
    
    plt.tight_layout()
    plt.show()
    
    return dict(zip(model_names, values))
```

## 17. How do you evaluate models with custom metrics?

**Answer:**
```python
def custom_metric_top_k_accuracy(outputs, targets, k=5):
    """Calculate top-k accuracy"""
    _, top_k_predictions = torch.topk(outputs, k, dim=1)
    correct = torch.zeros(targets.size(0), device=targets.device)
    
    for i in range(k):
        correct += (top_k_predictions[:, i] == targets).float()
    
    return correct.mean().item()

def custom_metric_mean_average_precision(outputs, targets, k=10):
    """Simplified MAP@k calculation"""
    # This is a simplified version - real implementation would be more complex
    _, top_k_predictions = torch.topk(outputs, k, dim=1)
    correct = (top_k_predictions == targets.unsqueeze(1)).float()
    return correct.mean().item()

def evaluate_with_custom_metrics(model, data_loader, device, custom_metrics=None):
    if custom_metrics is None:
        custom_metrics = {
            'top_1_accuracy': lambda outputs, targets: custom_metric_top_k_accuracy(outputs, targets, 1),
            'top_5_accuracy': lambda outputs, targets: custom_metric_top_k_accuracy(outputs, targets, 5),
        }
    
    model.eval()
    all_outputs = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            
            all_outputs.append(outputs)
            all_targets.append(targets)
    
    # Concatenate all outputs and targets
    all_outputs = torch.cat(all_outputs, dim=0)
    all_targets = torch.cat(all_targets, dim=0)
    
    # Calculate standard metrics
    _, predicted = torch.max(all_outputs, 1)
    accuracy = (predicted == all_targets).float().mean().item()
    
    # Calculate custom metrics
    custom_results = {}
    for metric_name, metric_func in custom_metrics.items():
        custom_results[metric_name] = metric_func(all_outputs, all_targets)
    
    return {
        'accuracy': accuracy,
        'custom_metrics': custom_results,
        'outputs': all_outputs.cpu().numpy(),
        'targets': all_targets.cpu().numpy()
    }
```

## 18. How do you implement evaluation with error analysis?

**Answer:**
```python
def error_analysis(model, data_loader, device, dataset_indices=None):
    model.eval()
    errors = []
    correct_predictions = []
    
    with torch.no_grad():
        for batch_idx, (inputs, targets) in enumerate(data_loader):
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            _, predicted = torch.max(outputs, 1)
            
            # Check each sample in batch
            for i in range(len(targets)):
                sample_idx = batch_idx * data_loader.batch_size + i
                if predicted[i] != targets[i]:
                    # Error case
                    confidence = torch.softmax(outputs[i], dim=0)[predicted[i]].item()
                    errors.append({
                        'index': sample_idx if dataset_indices is None else dataset_indices[sample_idx],
                        'true_label': targets[i].item(),
                        'predicted_label': predicted[i].item(),
                        'confidence': confidence,
                        'top_probs': torch.softmax(outputs[i], dim=0).cpu().numpy()
                    })
                else:
                    # Correct case
                    confidence = torch.softmax(outputs[i], dim=0)[predicted[i]].item()
                    correct_predictions.append({
                        'index': sample_idx if dataset_indices is None else dataset_indices[sample_idx],
                        'true_label': targets[i].item(),
                        'predicted_label': predicted[i].item(),
                        'confidence': confidence
                    })
    
    # Analyze errors
    error_analysis = {
        'total_errors': len(errors),
        'total_correct': len(correct_predictions),
        'error_rate': len(errors) / (len(errors) + len(correct_predictions)),
        'errors_by_class': {},
        'confidence_analysis': {
            'error_confidences': [e['confidence'] for e in errors],
            'correct_confidences': [c['confidence'] for c in correct_predictions]
        }
    }
    
    # Group errors by true class
    for error in errors:
        true_class = error['true_label']
        if true_class not in error_analysis['errors_by_class']:
            error_analysis['errors_by_class'][true_class] = []
        error_analysis['errors_by_class'][true_class].append(error)
    
    return error_analysis, errors, correct_predictions
```

## 19. How do you evaluate models with uncertainty quantification?

**Answer:**
```python
def evaluate_with_uncertainty(model, data_loader, device, n_samples=10):
    """Evaluate model with Monte Carlo dropout for uncertainty estimation"""
    model.train()  # Enable dropout for uncertainty estimation
    
    all_predictions = []
    all_uncertainties = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            # Multiple forward passes with dropout enabled
            predictions = []
            for _ in range(n_samples):
                outputs = model(inputs)
                probs = torch.softmax(outputs, dim=1)
                predictions.append(probs)
            
            # Stack predictions
            predictions = torch.stack(predictions)  # [n_samples, batch_size, n_classes]
            
            # Calculate mean prediction
            mean_prediction = predictions.mean(dim=0)
            _, predicted = torch.max(mean_prediction, 1)
            
            # Calculate uncertainty (variance across samples)
            uncertainty = predictions.var(dim=0).mean(dim=1)  # Variance of predictions
            
            all_predictions.extend(predicted.cpu().numpy())
            all_uncertainties.extend(uncertainty.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    return {
        'predictions': all_predictions,
        'uncertainties': all_uncertainties,
        'targets': all_targets,
        'mean_uncertainty': np.mean(all_uncertainties),
        'uncertainty_by_correctness': {
            'correct': [u for i, u in enumerate(all_uncertainties) 
                       if all_predictions[i] == all_targets[i]],
            'incorrect': [u for i, u in enumerate(all_uncertainties) 
                         if all_predictions[i] != all_targets[i]]
        }
    }
```

## 20. How do you implement evaluation with model calibration?

**Answer:**
```python
def evaluate_calibration(model, data_loader, device, n_bins=10):
    """Evaluate model calibration using reliability diagram"""
    model.eval()
    all_probabilities = []
    all_predictions = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            probabilities = torch.softmax(outputs, dim=1)
            _, predicted = torch.max(outputs, 1)
            
            all_probabilities.extend(probabilities.cpu().numpy())
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    # Convert to numpy arrays
    all_probabilities = np.array(all_probabilities)
    all_predictions = np.array(all_predictions)
    all_targets = np.array(all_targets)
    
    # For binary classification, take probabilities of positive class
    if all_probabilities.shape[1] == 2:
        confidences = all_probabilities[:, 1]
    else:
        # For multi-class, take maximum probability
        confidences = np.max(all_probabilities, axis=1)
    
    # Calculate accuracy in each bin
    bin_boundaries = np.linspace(0, 1, n_bins + 1)
    bin_lowers = bin_boundaries[:-1]
    bin_uppers = bin_boundaries[1:]
    
    accuracies = []
    confidences_in_bins = []
    prop_samples = []
    
    for bin_lower, bin_upper in zip(bin_lowers, bin_uppers):
        # Find samples in this bin
        in_bin = (confidences > bin_lower) & (confidences <= bin_upper)
        prop_in_bin = np.mean(in_bin)
        
        if prop_in_bin > 0:
            accuracy_in_bin = np.mean(all_targets[in_bin] == all_predictions[in_bin])
            avg_confidence_in_bin = np.mean(confidences[in_bin])
        else:
            accuracy_in_bin = 0
            avg_confidence_in_bin = 0
        
        accuracies.append(accuracy_in_bin)
        confidences_in_bins.append(avg_confidence_in_bin)
        prop_samples.append(prop_in_bin)
    
    # Calculate Expected Calibration Error (ECE)
    ece = np.sum(np.array(prop_samples) * 
                np.abs(np.array(accuracies) - np.array(confidences_in_bins)))
    
    return {
        'accuracies': accuracies,
        'confidences': confidences_in_bins,
        'proportions': prop_samples,
        'ece': ece,
        'reliability_diagram_data': {
            'bin_boundaries': bin_boundaries,
            'accuracies': accuracies,
            'confidences': confidences_in_bins
        }
    }

def plot_reliability_diagram(ece_data):
    """Plot reliability diagram"""
    plt.figure(figsize=(8, 8))
    plt.plot([0, 1], [0, 1], 'k--', label='Perfectly calibrated')
    plt.plot(ece_data['confidences'], ece_data['accuracies'], 'o-', 
             label=f'Model (ECE={ece_data["ece"]:.3f})')
    
    # Add histogram
    plt.bar(ece_data['reliability_diagram_data']['bin_boundaries'][:-1],
            ece_data['proportions'], width=0.1, alpha=0.3, 
            label='Fraction of samples')
    
    plt.xlabel('Confidence')
    plt.ylabel('Accuracy')
    plt.title('Reliability Diagram')
    plt.legend()
    plt.grid(True)
    plt.show()
```

## 21. How do you evaluate models with time-based metrics for streaming data?

**Answer:**
```python
import time

def evaluate_streaming_model(model, data_stream, device, window_size=1000):
    """Evaluate model performance on streaming data with time-based metrics"""
    model.eval()
    
    # Metrics tracking
    window_predictions = []
    window_targets = []
    processing_times = []
    throughput_rates = []
    
    start_time = time.time()
    samples_processed = 0
    
    for batch_idx, (inputs, targets) in enumerate(data_stream):
        batch_start_time = time.time()
        
        # Process batch
        inputs, targets = inputs.to(device), targets.to(device)
        
        with torch.no_grad():
            outputs = model(inputs)
            _, predicted = torch.max(outputs, 1)
        
        batch_processing_time = time.time() - batch_start_time
        processing_times.append(batch_processing_time)
        
        # Store predictions and targets
        window_predictions.extend(predicted.cpu().numpy())
        window_targets.extend(targets.cpu().numpy())
        
        samples_processed += len(targets)
        
        # Calculate throughput (samples per second)
        current_time = time.time()
        throughput = samples_processed / (current_time - start_time)
        throughput_rates.append(throughput)
        
        # Calculate window metrics when window is full
        if len(window_predictions) >= window_size:
            window_accuracy = accuracy_score(window_targets, window_predictions)
            avg_processing_time = np.mean(processing_times[-len(window_predictions)//len(targets):])
            
            print(f"Window {batch_idx//len(window_predictions)*len(targets) + 1}: "
                  f"Accuracy: {window_accuracy:.4f}, "
                  f"Avg Processing Time: {avg_processing_time:.4f}s, "
                  f"Throughput: {throughput:.2f} samples/sec")
            
            # Reset window
            window_predictions = []
            window_targets = []
    
    # Final metrics
    total_time = time.time() - start_time
    overall_throughput = samples_processed / total_time
    overall_avg_processing_time = np.mean(processing_times)
    
    return {
        'total_samples': samples_processed,
        'total_time': total_time,
        'overall_throughput': overall_throughput,
        'avg_processing_time': overall_avg_processing_time,
        'throughput_rates': throughput_rates,
        'processing_times': processing_times
    }
```

## 22. How do you implement evaluation with adversarial robustness testing?

**Answer:**
```python
def generate_fgsm_adversarial_example(model, x, y, epsilon=0.01):
    """Generate FGSM adversarial example"""
    x_adv = x.clone().detach().requires_grad_(True)
    
    output = model(x_adv)
    loss = torch.nn.functional.cross_entropy(output, y)
    loss.backward()
    
    # FGSM attack
    x_adv = x_adv + epsilon * x_adv.grad.sign()
    x_adv = torch.clamp(x_adv, 0, 1)
    
    return x_adv.detach()

def evaluate_adversarial_robustness(model, data_loader, device, epsilons=[0.01, 0.05, 0.1]):
    """Evaluate model robustness against adversarial attacks"""
    model.eval()
    robustness_results = {}
    
    for epsilon in epsilons:
        correct_clean = 0
        correct_adv = 0
        total = 0
        
        with torch.no_grad():
            for inputs, targets in data_loader:
                inputs, targets = inputs.to(device), targets.to(device)
                
                # Clean accuracy
                outputs_clean = model(inputs)
                _, predicted_clean = torch.max(outputs_clean, 1)
                correct_clean += (predicted_clean == targets).sum().item()
                
                # Generate adversarial examples
                inputs_adv = generate_fgsm_adversarial_example(model, inputs, targets, epsilon)
                
                # Adversarial accuracy
                outputs_adv = model(inputs_adv)
                _, predicted_adv = torch.max(outputs_adv, 1)
                correct_adv += (predicted_adv == targets).sum().item()
                
                total += targets.size(0)
        
        clean_accuracy = 100 * correct_clean / total
        adv_accuracy = 100 * correct_adv / total
        robustness = 100 * (correct_clean - correct_adv) / correct_clean if correct_clean > 0 else 0
        
        robustness_results[epsilon] = {
            'clean_accuracy': clean_accuracy,
            'adversarial_accuracy': adv_accuracy,
            'robustness_drop': robustness
        }
        
        print(f"Epsilon {epsilon}: Clean {clean_accuracy:.2f}%, "
              f"Adversarial {adv_accuracy:.2f}%, "
              f"Drop {robustness:.2f}%")
    
    return robustness_results
```

## 23. How do you evaluate models with fairness metrics?

**Answer:**
```python
def evaluate_fairness(model, data_loader, device, sensitive_attribute_idx=0):
    """Evaluate model fairness across different groups"""
    model.eval()
    
    # Collect predictions and sensitive attributes
    all_predictions = []
    all_targets = []
    all_sensitive_attrs = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            sensitive_attrs = inputs[:, sensitive_attribute_idx]  # Assuming sensitive attr is a feature
            inputs, targets = inputs.to(device), targets.to(device)
            
            outputs = model(inputs)
            _, predicted = torch.max(outputs, 1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
            all_sensitive_attrs.extend(sensitive_attrs.cpu().numpy())
    
    # Convert to numpy arrays
    predictions = np.array(all_predictions)
    targets = np.array(all_targets)
    sensitive_attrs = np.array(all_sensitive_attrs)
    
    # Unique groups
    unique_groups = np.unique(sensitive_attrs)
    
    fairness_metrics = {}
    
    for group in unique_groups:
        group_mask = sensitive_attrs == group
        group_predictions = predictions[group_mask]
        group_targets = targets[group_mask]
        
        if len(group_targets) > 0:
            group_accuracy = accuracy_score(group_targets, group_predictions)
            group_precision = precision_score(group_targets, group_predictions, average='weighted', zero_division=0)
            group_recall = recall_score(group_targets, group_predictions, average='weighted', zero_division=0)
            
            fairness_metrics[f'group_{group}'] = {
                'accuracy': group_accuracy,
                'precision': group_precision,
                'recall': group_recall,
                'sample_count': len(group_targets)
            }
    
    # Calculate fairness measures
    accuracies = [metrics['accuracy'] for metrics in fairness_metrics.values()]
    max_diff_accuracy = max(accuracies) - min(accuracies) if len(accuracies) > 1 else 0
    
    fairness_metrics['overall'] = {
        'max_accuracy_difference': max_diff_accuracy,
        'groups_count': len(unique_groups)
    }
    
    return fairness_metrics
```

## 24. How do you implement evaluation with memory and computational efficiency metrics?

**Answer:**
```python
import psutil
import torch.cuda as cuda

def evaluate_efficiency(model, data_loader, device):
    """Evaluate model efficiency in terms of memory and computation"""
    model.eval()
    
    # Memory usage before
    if device.type == 'cuda':
        torch.cuda.reset_peak_memory_stats()
        mem_before = torch.cuda.memory_allocated()
    else:
        process = psutil.Process()
        mem_before = process.memory_info().rss / 1024 / 1024  # MB
    
    # Timing
    start_time = time.time()
    total_samples = 0
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            # Forward pass timing
            batch_start = time.time()
            outputs = model(inputs)
            batch_time = time.time() - batch_start
            
            total_samples += len(targets)
    
    total_time = time.time() - start_time
    throughput = total_samples / total_time
    
    # Memory usage after
    if device.type == 'cuda':
        mem_after = torch.cuda.memory_allocated()
        peak_mem = torch.cuda.max_memory_allocated()
        mem_used = (mem_after - mem_before) / 1024 / 1024  # MB
        peak_mem_used = peak_mem / 1024 / 1024  # MB
    else:
        mem_after = process.memory_info().rss / 1024 / 1024  # MB
        mem_used = mem_after - mem_before
        peak_mem_used = mem_used  # Approximation for CPU
    
    # Model size
    model_size = sum(p.numel() * p.element_size() for p in model.parameters()) / 1024 / 1024  # MB
    
    return {
        'total_time': total_time,
        'throughput_samples_per_sec': throughput,
        'memory_used_mb': mem_used,
        'peak_memory_mb': peak_mem_used,
        'model_size_mb': model_size,
        'avg_time_per_sample': total_time / total_samples,
        'samples_processed': total_samples
    }
```

## 25. How do you evaluate models with statistical significance testing?

**Answer:**
```python
from scipy import stats

def statistical_significance_test(model1_preds, model2_preds, targets, test_type='ttest'):
    """Perform statistical significance test between two models"""
    
    # Calculate accuracies for each sample (1 for correct, 0 for incorrect)
    model1_correct = (np.array(model1_preds) == np.array(targets)).astype(int)
    model2_correct = (np.array(model2_preds) == np.array(targets)).astype(int)
    
    # Difference in correctness
    differences = model1_correct - model2_correct
    
    if test_type == 'ttest':
        # Paired t-test
        t_stat, p_value = stats.ttest_rel(model1_correct, model2_correct)
        test_name = "Paired t-test"
    elif test_type == 'wilcoxon':
        # Wilcoxon signed-rank test
        t_stat, p_value = stats.wilcoxon(model1_correct, model2_correct)
        test_name = "Wilcoxon signed-rank test"
    elif test_type == 'mcnemar':
        # McNemar's test for binary classification
        # Contingency table
        both_correct = np.sum((model1_correct == 1) & (model2_correct == 1))
        model1_correct_only = np.sum((model1_correct == 1) & (model2_correct == 0))
        model2_correct_only = np.sum((model1_correct == 0) & (model2_correct == 1))
        both_incorrect = np.sum((model1_correct == 0) & (model2_correct == 0))
        
        # McNemar's test statistic
        if (model1_correct_only + model2_correct_only) > 0:
            mcnemar_stat = (abs(model1_correct_only - model2_correct_only) - 1) ** 2 / (model1_correct_only + model2_correct_only)
            p_value = 1 - stats.chi2.cdf(mcnemar_stat, df=1)
        else:
            mcnemar_stat = 0
            p_value = 1.0
        t_stat = mcnemar_stat
        test_name = "McNemar's test"
    else:
        raise ValueError("test_type must be 'ttest', 'wilcoxon', or 'mcnemar'")
    
    # Effect size (Cohen's d for t-test)
    if test_type == 'ttest':
        mean_diff = np.mean(differences)
        std_diff = np.std(differences)
        effect_size = mean_diff / std_diff if std_diff > 0 else 0
    else:
        effect_size = None
    
    return {
        'test_name': test_name,
        'test_statistic': t_stat,
        'p_value': p_value,
        'is_significant': p_value < 0.05,
        'effect_size': effect_size,
        'model1_accuracy': np.mean(model1_correct),
        'model2_accuracy': np.mean(model2_correct),
        'accuracy_difference': np.mean(model1_correct) - np.mean(model2_correct)
    }
```

## 26. How do you implement evaluation with detailed per-class analysis?

**Answer:**
```python
def detailed_per_class_analysis(model, data_loader, device, class_names=None):
    """Perform detailed per-class analysis"""
    model.eval()
    
    all_predictions = []
    all_targets = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            _, predicted = torch.max(outputs, 1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
    
    # Convert to numpy arrays
    predictions = np.array(all_predictions)
    targets = np.array(all_targets)
    
    # Unique classes
    unique_classes = np.unique(np.concatenate([targets, predictions]))
    n_classes = len(unique_classes)
    
    if class_names is None:
        class_names = [f"Class_{i}" for i in unique_classes]
    
    # Calculate per-class metrics
    per_class_metrics = {}
    
    for i, class_id in enumerate(unique_classes):
        # Create binary masks
        class_mask = targets == class_id
        predicted_class_mask = predictions == class_id
        
        # True/False Positives/Negatives
        tp = np.sum((targets == class_id) & (predictions == class_id))
        fp = np.sum((targets != class_id) & (predictions == class_id))
        fn = np.sum((targets == class_id) & (predictions != class_id))
        tn = np.sum((targets != class_id) & (predictions != class_id))
        
        # Calculate metrics
        precision = tp / (tp + fp) if (tp + fp) > 0 else 0
        recall = tp / (tp + fn) if (tp + fn) > 0 else 0
        f1 = 2 * (precision * recall) / (precision + recall) if (precision + recall) > 0 else 0
        specificity = tn / (tn + fp) if (tn + fp) > 0 else 0
        accuracy = (tp + tn) / (tp + tn + fp + fn) if (tp + tn + fp + fn) > 0 else 0
        
        # Support (number of actual samples)
        support = np.sum(class_mask)
        
        per_class_metrics[class_names[i]] = {
            'precision': precision,
            'recall': recall,
            'f1_score': f1,
            'specificity': specificity,
            'accuracy': accuracy,
            'support': int(support),
            'tp': int(tp),
            'fp': int(fp),
            'fn': int(fn),
            'tn': int(tn)
        }
    
    # Overall metrics
    overall_accuracy = accuracy_score(targets, predictions)
    macro_precision = np.mean([metrics['precision'] for metrics in per_class_metrics.values()])
    macro_recall = np.mean([metrics['recall'] for metrics in per_class_metrics.values()])
    macro_f1 = np.mean([metrics['f1_score'] for metrics in per_class_metrics.values()])
    
    return {
        'per_class_metrics': per_class_metrics,
        'overall_metrics': {
            'accuracy': overall_accuracy,
            'macro_precision': macro_precision,
            'macro_recall': macro_recall,
            'macro_f1': macro_f1
        },
        'confusion_matrix': confusion_matrix(targets, predictions)
    }
```

## 27. How do you evaluate models with temporal consistency for time-series data?

**Answer:**
```python
def evaluate_temporal_consistency(model, time_series_data_loader, device, sequence_length):
    """Evaluate temporal consistency in time-series predictions"""
    model.eval()
    
    all_predictions = []
    all_targets = []
    consistency_scores = []
    
    with torch.no_grad():
        for sequences, targets in time_series_data_loader:
            sequences, targets = sequences.to(device), targets.to(device)
            
            # Get predictions for full sequences
            outputs = model(sequences)
            _, predicted = torch.max(outputs, 1)
            
            all_predictions.extend(predicted.cpu().numpy())
            all_targets.extend(targets.cpu().numpy())
            
            # Check temporal consistency
            # For each sequence, check if predictions are consistent across time
            batch_size = sequences.size(0)
            for i in range(batch_size):
                sequence = sequences[i]  # [seq_len, features]
                # Get predictions for overlapping windows
                window_predictions = []
                for j in range(sequence_length - 5):  # 5 is window overlap
                    window = sequence[j:j+5].unsqueeze(0)  # [1, 5, features]
                    window_output = model(window)
                    _, window_pred = torch.max(window_output, 1)
                    window_predictions.append(window_pred.item())
                
                # Calculate consistency (how often predictions change)
                if len(window_predictions) > 1:
                    changes = sum(1 for k in range(len(window_predictions)-1) 
                                if window_predictions[k] != window_predictions[k+1])
                    consistency = 1.0 - (changes / (len(window_predictions) - 1))
                    consistency_scores.append(consistency)
    
    # Calculate metrics
    accuracy = accuracy_score(all_targets, all_predictions)
    avg_consistency = np.mean(consistency_scores) if consistency_scores else 0
    
    return {
        'accuracy': accuracy,
        'temporal_consistency': avg_consistency,
        'consistency_scores': consistency_scores,
        'predictions': all_predictions,
        'targets': all_targets
    }
```

## 28. How do you implement evaluation with domain adaptation metrics?

**Answer:**
```python
def evaluate_domain_adaptation(model, source_loader, target_loader, device):
    """Evaluate model performance across source and target domains"""
    model.eval()
    
    def evaluate_domain(data_loader, domain_name):
        all_predictions = []
        all_targets = []
        
        with torch.no_grad():
            for inputs, targets in data_loader:
                inputs, targets = inputs.to(device), targets.to(device)
                outputs = model(inputs)
                _, predicted = torch.max(outputs, 1)
                
                all_predictions.extend(predicted.cpu().numpy())
                all_targets.extend(targets.cpu().numpy())
        
        accuracy = accuracy_score(all_targets, all_predictions)
        f1 = f1_score(all_targets, all_predictions, average='weighted')
        
        return {
            'accuracy': accuracy,
            'f1_score': f1,
            'predictions': all_predictions,
            'targets': all_targets,
            'domain': domain_name
        }
    
    # Evaluate on both domains
    source_results = evaluate_domain(source_loader, 'source')
    target_results = evaluate_domain(target_loader, 'target')
    
    # Calculate domain gap
    domain_gap = abs(source_results['accuracy'] - target_results['accuracy'])
    
    return {
        'source_domain': source_results,
        'target_domain': target_results,
        'domain_gap': domain_gap,
        'adaptation_score': 1.0 - domain_gap  # Higher is better adaptation
    }
```

## 29. How do you evaluate models with interpretability metrics?

**Answer:**
```python
def evaluate_interpretability(model, data_loader, device, explanation_method='gradient'):
    """Evaluate model interpretability"""
    model.eval()
    
    interpretability_scores = []
    faithfulness_scores = []
    
    with torch.no_grad():
        for inputs, targets in data_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            # Get model predictions
            outputs = model(inputs)
            _, predicted = torch.max(outputs, 1)
            
            batch_size = inputs.size(0)
            for i in range(batch_size):
                input_sample = inputs[i:i+1]
                target_sample = targets[i:i+1]
                predicted_sample = predicted[i:i+1]
                
                # Generate explanation (simplified)
                if explanation_method == 'gradient':
                    # Gradient-based explanation
                    input_sample.requires_grad_(True)
                    output = model(input_sample)
                    output[0, predicted_sample].backward()
                    explanation = input_sample.grad.abs().mean(dim=1)  # [H, W] for images
                else:
                    explanation = torch.rand_like(input_sample[0])  # Placeholder
                
                # Calculate interpretability metrics
                # Sparsity: how focused the explanation is
                explanation_flat = explanation.flatten()
                sparsity = 1.0 - (explanation_flat > explanation_flat.mean()).float().mean()
                
                # Faithfulness: correlation between explanation and actual importance
                # This is a simplified version - real implementation would be more complex
                faithfulness = torch.rand(1).item()  # Placeholder
                
                interpretability_scores.append(sparsity.item())
                faithfulness_scores.append(faithfulness)
    
    return {
        'avg_sparsity': np.mean(interpretability_scores),
        'avg_faithfulness': np.mean(faithfulness_scores),
        'interpretability_scores': interpretability_scores,
        'faithfulness_scores': faithfulness_scores,
        'interpretability_score': np.mean(interpretability_scores),
        'faithfulness_score': np.mean(faithfulness_scores)
    }
```

## 30. How do you implement comprehensive model evaluation with reporting?

**Answer:**
```python
def comprehensive_model_evaluation(model, train_loader, val_loader, test_loader, 
                                 device, class_names=None, save_report=True, 
                                 report_dir='./reports'):
    """Comprehensive model evaluation with detailed reporting"""
    
    # Create report directory
    os.makedirs(report_dir, exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    report_file = f"{report_dir}/model_evaluation_{timestamp}.json"
    
    evaluation_results = {
        'timestamp': timestamp,
        'model_info': {
            'model_class': model.__class__.__name__,
            'num_parameters': sum(p.numel() for p in model.parameters()),
            'num_trainable_parameters': sum(p.numel() for p in model.parameters() if p.requires_grad)
        }
    }
    
    # Evaluate on all datasets
    datasets = {
        'train': train_loader,
        'validation': val_loader,
        'test': test_loader
    }
    
    for dataset_name, data_loader in datasets.items():
        print(f"Evaluating on {dataset_name} set...")
        
        # Basic metrics
        results = comprehensive_evaluation(model, data_loader, device, task_type='classification')
        
        # Detailed per-class analysis
        per_class_results = detailed_per_class_analysis(model, data_loader, device, class_names)
        
        # Efficiency metrics
        efficiency_results = evaluate_efficiency(model, data_loader, device)
        
        evaluation_results[dataset_name] = {
            'basic_metrics': results,
            'per_class_analysis': per_class_results,
            'efficiency_metrics': efficiency_results
        }
    
    # Cross-dataset comparison
    if test_loader is not None:
        evaluation_results['cross_dataset_comparison'] = {
            'train_test_gap': abs(evaluation_results['train']['basic_metrics']['accuracy'] - 
                                evaluation_results['test']['basic_metrics']['accuracy']),
            'val_test_gap': abs(evaluation_results['validation']['basic_metrics']['accuracy'] - 
                              evaluation_results['test']['basic_metrics']['accuracy'])
        }
    
    # Save report
    if save_report:
        with open(report_file, 'w') as f:
            json.dump(evaluation_results, f, indent=2)
        print(f"Evaluation report saved to {report_file}")
    
    return evaluation_results

def generate_evaluation_summary(evaluation_results):
    """Generate human-readable summary of evaluation results"""
    summary = []
    summary.append("=== MODEL EVALUATION SUMMARY ===\n")
    
    # Model info
    model_info = evaluation_results['model_info']
    summary.append(f"Model: {model_info['model_class']}")
    summary.append(f"Parameters: {model_info['num_parameters']:,}")
    summary.append(f"Trainable Parameters: {model_info['num_trainable_parameters']:,}\n")
    
    # Performance metrics
    for dataset in ['train', 'validation', 'test']:
        if dataset in evaluation_results:
            metrics = evaluation_results[dataset]['basic_metrics']
            summary.append(f"{dataset.upper()} SET:")
            summary.append(f"  Accuracy: {metrics['accuracy']:.4f}")
            summary.append(f"  F1-Score: {metrics['f1_score']:.4f}")
            summary.append(f"  AUC: {metrics['auc']:.4f}\n")
    
    # Gaps
    if 'cross_dataset_comparison' in evaluation_results:
        gaps = evaluation_results['cross_dataset_comparison']
        summary.append("PERFORMANCE GAPS:")
        summary.append(f"  Train-Test Gap: {gaps['train_test_gap']:.4f}")
        summary.append(f"  Val-Test Gap: {gaps['val_test_gap']:.4f}\n")
    
    # Efficiency
    if 'test' in evaluation_results:
        efficiency = evaluation_results['test']['efficiency_metrics']
        summary.append("EFFICIENCY METRICS:")
        summary.append(f"  Throughput: {efficiency['throughput_samples_per_sec']:.2f} samples/sec")
        summary.append(f"  Memory Usage: {efficiency['memory_used_mb']:.2f} MB")
        summary.append(f"  Model Size: {efficiency['model_size_mb']:.2f} MB\n")
    
    return "\n".join(summary)
```

## 31. How do you implement evaluation with model checkpoint selection?

**Answer:**
```python
def select_best_checkpoint(model, checkpoint_dir, val_loader, device, 
                          metric='accuracy', mode='max'):
    """Select best model checkpoint based on validation performance"""
    
    # Get all checkpoint files
    checkpoint_files = [f for f in os.listdir(checkpoint_dir) 
                       if f.endswith('.pth') or f.endswith('.pt')]
    
    best_score = float('-inf') if mode == 'max' else float('inf')
    best_checkpoint = None
    checkpoint_scores = {}
    
    for checkpoint_file in checkpoint_files:
        checkpoint_path = os.path.join(checkpoint_dir, checkpoint_file)
        
        # Load checkpoint
        checkpoint = torch.load(checkpoint_path, map_location=device)
        model.load_state_dict(checkpoint['model_state_dict'])
        
        # Evaluate on validation set
        val_results = validate_model(model, val_loader, 
                                   torch.nn.CrossEntropyLoss(), device)
        
        score = val_results[metric] if metric in val_results else val_results['accuracy']
        checkpoint_scores[checkpoint_file] = score
        
        # Update best checkpoint
        if (mode == 'max' and score > best_score) or (mode == 'min' and score < best_score):
            best_score = score
            best_checkpoint = checkpoint_file
    
    # Load best checkpoint
    if best_checkpoint:
        best_path = os.path.join(checkpoint_dir, best_checkpoint)
        best_checkpoint_data = torch.load(best_path, map_location=device)
        model.load_state_dict(best_checkpoint_data['model_state_dict'])
        
        print(f"Loaded best checkpoint: {best_checkpoint}")
        print(f"Best {metric}: {best_score:.4f}")
    
    return {
        'best_checkpoint': best_checkpoint,
        'best_score': best_score,
        'all_scores': checkpoint_scores,
        'checkpoint_path': os.path.join(checkpoint_dir, best_checkpoint) if best_checkpoint else None
    }
```