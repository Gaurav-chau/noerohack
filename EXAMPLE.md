# Example Implementation Guide

This file demonstrates how to implement code based on markdown specifications.

## Example: Simple Data Analysis Task

### Given this markdown specification:

```markdown
# Data Analysis Task

Create a Python program that:
1. Loads a CSV file
2. Performs basic statistical analysis
3. Visualizes the results
```

### Implementation in Python (main.py):

```python
import pandas as pd
import matplotlib.pyplot as plt

def load_data(filepath):
    """Load data from CSV file."""
    return pd.read_csv(filepath)

def analyze_data(df):
    """Perform statistical analysis."""
    return df.describe()

def visualize_data(df):
    """Create visualizations."""
    df.plot(kind='bar')
    plt.show()

def main():
    df = load_data('data.csv')
    stats = analyze_data(df)
    print(stats)
    visualize_data(df)

if __name__ == "__main__":
    main()
```

### Implementation in Jupyter Notebook:

Create cells for:
1. Imports and setup
2. Data loading
3. Analysis
4. Visualization
5. Results interpretation

## Process

1. **Read** the markdown file with requirements
2. **Plan** the implementation structure
3. **Code** in Python or Jupyter as appropriate
4. **Test** the implementation
5. **Document** the usage

## Tips

- Use Python scripts (.py) for production code, CLI tools, or automated tasks
- Use Jupyter notebooks (.ipynb) for data analysis, exploration, or teaching
- Always include proper error handling
- Add docstrings to functions
- Follow PEP 8 style guidelines
