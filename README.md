# MacroTracker

MacroTracker is a notebook-based meal analysis prototype. It accepts a photo of a meal and a user's nutrition goal, then runs a three-stage pipeline:

1. **Food detection**: a Faster R-CNN model identifies food items in the image.
2. **Nutrition estimation**: detected labels are matched to USDA FoodData Central records and scaled to an estimated portion size.
3. **Dietary coaching**: a Qwen instruct model produces feedback about calories, protein, carbohydrates, fat, and the user's goal. A local LoRA adapter is included for this stage.

The project is intended for experimentation and research. Portion sizes are estimated from image bounding-box area, so the nutrition values should not be treated as medical or dietary advice.

## Requirements

- Windows, macOS, or Linux
- Python **3.13** (the notebooks were saved with Python 3.13.2)
- Git
- A few GB of disk space for Python packages and the Hugging Face Qwen model
- NVIDIA CUDA is recommended for faster model loading and detector training, but the code can fall back to CPU

## Project structure

```text
.
├── app/
│   ├── executor.ipynb              # End-to-end meal analysis
│   ├── image_scaner.ipynb          # Dataset loader, detector, training, inference
│   ├── nutritions_estimater.ipynb  # USDA lookup and macro calculation
│   ├── fine_tuning.ipynb           # Qwen loading and optional LoRA fine-tuning
│   └── models/
│       └── lora_coaching_adapter/  # Included coaching adapter
├── example_images/                 # Example meal images
├── image_detection_data/           # COCO annotations and training images
├── nutrition_data/                 # USDA CSV files
├── requirements.txt
└── resources.txt                   # Dataset source links
```

## Installation

From the repository root, create and activate a virtual environment:

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

If PowerShell blocks activation, run this once in the current PowerShell session or activate the environment from another shell:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Register the environment as a Jupyter kernel:

```powershell
python -m ipykernel install --user --name macrotracker --display-name "MacroTracker (.venv)"
```

In VS Code, install or enable the **Jupyter** extension, open a notebook under `app/`, and select the `MacroTracker (.venv)` kernel.

## Data and model assets

The repository already contains:

- COCO-format food annotations in `image_detection_data/annotations.json`
- Food detection images in `image_detection_data/images/`
- USDA-derived CSV files in `nutrition_data/`
- The LoRA coaching adapter in `app/models/lora_coaching_adapter/`

The source links for the food-recognition and USDA datasets are recorded in `resources.txt`.

The end-to-end notebook also expects this detector checkpoint:

```text
app/models/fasterrcnn_food2022.pth
```

That file is not currently included in the repository. Create it by following the detector training steps below, or provide a compatible checkpoint at that path. Without it, the notebook builds an untrained detector head and its food predictions will not be useful.

The first execution of the coaching stage downloads `Qwen/Qwen2.5-1.5B-Instruct` from Hugging Face. Internet access is required for that first download.

## Run the end-to-end pipeline

1. Open `app/image_scaner.ipynb`.
2. Run the setup, dataset, model, and training cells. The notebook trains for three epochs and saves:

   ```text
   app/models/fasterrcnn_food2022.pth
   ```

   Adjust `batch_size`, `num_workers`, and `epochs` if your hardware has limited memory. CPU training may be very slow.

3. Open `app/executor.ipynb` and run the import cell followed by the function-definition cell.
4. Call the pipeline with an image path relative to the `app/` directory and a user goal:

   ```python
   result = analyze_meal_from_image(
       user_image_path="../example_images/chiken_and_rice.png",
       user_goal="Fat Loss (Target: Under 500 kcal, High Protein)",
   )
   print(result)
   ```

Keep the notebook working directory set to `app/`. The executor uses relative paths such as `../nutrition_data` and `models/fasterrcnn_food2022.pth`.

The returned dictionary has this shape:

```python
{
    "source": "local_llm",
    "data": "generated dietary coaching text",
}
```

## Run individual stages

- **Food detection**: use `image_scaner.ipynb` to load the COCO dataset, train the detector, or run `predict_meal_components(...)` on an image.
- **Nutrition estimation**: use `nutritions_estimater.ipynb` to load the USDA tables and run `calculate_meal_nutrition(...)` on detector output.
- **Coaching**: use `fine_tuning.ipynb` to load the Qwen model and generate feedback. Fine-tuning is optional; the notebook's example uses the base instruct model by default. Set `run_training=True` only when you intentionally want to recreate the adapter.

## Optional OpenAI fallback

The current `run_resilient_coaching_pipeline(...)` implementation uses the local model and reports an error if local generation fails. It accepts `OPENAI_API_KEY` for future or extended fallback behavior, but setting the variable is not required for the current notebook workflow.

## Troubleshooting

- **Import errors in `executor.ipynb`**: make sure the notebook is running with `app/` as its working directory and that `import-ipynb` is installed.
- **Missing checkpoint**: train `image_scaner.ipynb` first and confirm `app/models/fasterrcnn_food2022.pth` exists.
- **Slow or memory-heavy execution**: use a CUDA-enabled PyTorch environment and GPU, or expect CPU inference and Qwen loading to be slower.
- **No nutrition matches**: the nutrition stage uses fuzzy matching against the USDA descriptions and excludes unmatched items from the totals.
- **Hugging Face download problems**: confirm internet access and that the local Hugging Face cache has enough disk space.

## Data sources

See [`resources.txt`](resources.txt) for the Food Recognition 2022 dataset and USDA FoodData Central download links.
