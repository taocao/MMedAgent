# MMedAgent: Learning to Use Medical Tools with Multi-modal Agent

The first multimodal medical AI Agent incorporating a wide spectrum of tools to handle various medical
tasks across different modalities seamlessly.

[[Paper, EMNLP 2024 (Findings)](https://arxiv.org/abs/2407.02483)] [Demo](https://1cc0bf26516bc745fd.gradio.live/)  (*NOTE: This is a temporary link. Choose "merge_med_llava_3" in the dropdown menu on top left*)

Binxu Li, Tiankai Yan, Yuanting Pan, Jie Luo, Ruiyang Ji, Jiayuan Ding, Zhe Xu, Shilong Liu, Haoyu Dong*, Zihao Lin*, Yixin Wang* 

<div style="text-align: center;">
    <img src="imgs/mmedagent.jpg" alt="MMedAgent" style="width: 50%;"/>
    <img src="imgs/instruction-tuning-data.jpg" alt="Instruction Tuning Data" style="width: 50%;"/>
</div>

## Current Tool lists

| Task           | Tool                                     | Data Source                                                                                                                       | Imaging Modality                             |
|----------------|------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| VQA            | [LLaVA-Med](https://github.com/microsoft/LLaVA-Med/tree/main)                    | PMC article<br>*60K-IM*                                                                                                | MRI, CT, X-ray, Histology, Gross            |
| Classification | [BiomedCLIP](https://huggingface.co/microsoft/BiomedCLIP-PubMedBERT_256-vit_base_patch16_224)                       | PMC article<br>*60K-IM*                                                                                                         | MRI, CT, X-ray, Histology, Gross            |
| Grounding      | [Grounding DINO](https://github.com/IDEA-Research/GroundingDINO)                    | WORD, etc.*<br>                                                                                                                 | MRI, CT, X-ray, Histology                   |
| Segmentation with bounding-box prompts (Segmentation)    | [MedSAM](https://github.com/bowang-lab/MedSAM)                            | WORD, etc.*                                                                                                                      | MRI, CT, X-ray, Histology, Gross            |
| Segmentation with text prompts (G-Seg)        | [Grounding DINO](https://github.com/IDEA-Research/GroundingDINO)  + [MedSAM](https://github.com/bowang-lab/MedSAM)                  | WORD, etc.*                                                                                                                      | MRI, CT, X-ray, Histology                   |
| Medical report generation (MRG)            | [ChatCAD](https://github.com/zhaozh10/ChatCAD)                           | MIMIC-CXR                                                                                                               | X-ray                                        |
| Retrieval augmented generation (RAG)            | [ChatCAD+](https://github.com/zhaozh10/ChatCAD)                         | Merck Manual                                                                                                            | --                                           |

---

**Note**: ``--`` means that the RAG task only focuses on natural language without handling images. ``WORD, etc.*`` indicates various data sources including WORD, FLARE2021, BRATS, Montgomery County X-ray Set (MC), VinDr-CXR, and Cellseg.  


## Usage
1. Clone this repo and navigate to xxx folder
```
git clone https://github.com/Wangyixinxin/MMedAgent.git
```
2. Download ChatCAD Dependencies

- Please download the dependent checkpoints and JSON files for both [ChatCAD_R](ChatCAD_R) and [MMedAgent/src/ChatCAD_R](MMedAgent/src/ChatCAD_R).

- You can download from either the original ChatCAD [repo](https://github.com/zhaozh10/ChatCAD?tab=readme-ov-file) or from [Google Drive](https://drive.google.com/drive/folders/14OWwsFjphsjqT-nH9GHgf5Sy7f1aL9Lz?usp=sharing).
  
- Please save r2gcmn_mimic-cxr.pth and JFchexpert.pth in ChatCAD_R/weights/ and save annotation.json in ChatCAD_R/.

3. Download Model Checkpoint

   TBA

4. Download Tools
   
   - GroundingDINO
    ```
    cd MMedAgent/src
    git clone https://github.com/IDEA-Research/GroundingDINO.git
    ```
   - MedSAM
    ```
    cd MMedAgent/src
    git clone https://github.com/bowang-lab/MedSAM.git
    ```

6. Web UI Inference
- Create environment
```
cd MMedAgent
conda create -f environment.yml
```
- Run the following commands in separate terminals:

  - Launch controller
    ```
    python -m llava.serve.controller --host 0.0.0.0 --port 20001
    ```
  - Launch model worker
    ```
    python -m llava.serve.model_worker --host 0.0.0.0 --controller http://localhost:20001 --port 40000 --worker http://localhost:40000 --model-path <Your Model Path>
    ```
  - Launch tool workers
    ```
    python serve/grounding_dino_worker.py
    python serve/MedSAM_worker.py
    python serve/grounded_medsam_worker.py
    python serve/biomedclip_worker.py
    python serve/chatcad_G_worker.py
    python serve/chatcad_R_worker.py    
    ```
  - Launch gradio web server
    ```
    python llava/serve/gradio_web_server_mmedagent.py --controller http://localhost:20001 --model-list-mode reload
    ```
- You can now access the model in localhost:7860.
## Base Model Download

The model weights below are *delta* weights. The usage of LLaVA-Med checkpoints should comply with the base LLM's model license: [LLaMA](https://github.com/facebookresearch/llama/blob/main/MODEL_CARD.md).

We provide delta weights for LLaVA-Med and 3 LLaVA-Med models each finetuned on the 3 VQA datasets:

 Model Descriptions | Model Delta Weights | Size |
| --- | --- | ---: |
| LLaVA-Med | [llava_med_in_text_60k_ckpt2_delta.zip](https://hanoverprod.z21.web.core.windows.net/med_llava/models/llava_med_in_text_60k_ckpt2_delta.zip) | 11.06 GB |

Instructions:

1. Download the delta weights above and unzip.
1. Get the original LLaMA weights in the huggingface format by following the instructions [here](https://huggingface.co/docs/transformers/main/model_doc/llama).
1. Use the following scripts to get original LLaVA-Med weights by applying our delta. In the script below, set the --delta argument to the path of the unzipped delta weights directory from step 1.

```bash
python3 -m llava.model.apply_delta \
    --base /path/to/llama-7b \
    --target ./base_model \
    --delta /path/to/llava_med_delta_weights
```
## Train
train with lora:
```
deepspeed llava/train/train_mem.py \
    --lora_enable True --lora_r 128 --lora_alpha 256 --mm_projector_lr 2e-5 \
    --deepspeed ./scripts/zero2.json \
    --model_name_or_path ./base_model  \
    --version v1\
    --data_path ./train_data_json/example.jsonl \
    --image_folder ./train_images \
    --vision_tower openai/clip-vit-large-patch14-336 \
    --mm_projector_type mlp2x_gelu \
    --mm_vision_select_layer -2 \
    --mm_use_im_start_end False \
    --mm_use_im_patch_token False \
    --image_aspect_ratio pad \
    --group_by_modality_length False \
    --bf16 True \
    --output_dir ./checkpoints/output_lora_weights \
    --num_train_epochs 30 \
    --per_device_train_batch_size 12 \
    --per_device_eval_batch_size 1 \
    --gradient_accumulation_steps 2 \
    --evaluation_strategy "no" \
    --save_strategy "steps" \
    --save_steps 3000 \
    --save_total_limit 2 \
    --learning_rate 2e-4 \
    --weight_decay 0. \
    --warmup_ratio 0.03 \
    --lr_scheduler_type "cosine" \
    --logging_steps 1 \
    --tf32 True \
    --model_max_length 2048 \
    --gradient_checkpointing True \
    --dataloader_num_workers 4 \
    --lazy_preprocess True \
    --report_to wandb
```
or use [`tuning.sh`](https://github.com/Wangyixinxin/MMedAgent/blob/main/tuning.sh)
## Evaluation
### apply lora (if you enable lora when training)
```
CUDA_VISIBLE_DEVICES=0 python scripts/merge_lora_weights.py \
    --model-path ./checkpoints/output_lora_weights \
    --model-base ./base_model \
    --save-model-path ./llava_med_agent
```
or use [`merge.sh`](https://github.com/Wangyixinxin/MMedAgent/blob/main/merge.sh)
### Inference
```
CUDA_VISIBLE_DEVICES=0 python llava/eval/model_vqa.py \
    --model-path ./llava_med_agent \
    --question-file ./eval_data_json/eval_example.jsonl \
    --image-folder ./eval_images \
    --answers-file ./eval_data_json/output_agent_eval_example.jsonl \
    --temperature 0.2
```
or use [`eval.sh`](https://github.com/Wangyixinxin/MMedAgent/blob/main/eval.sh)
### GPT-4o inference
```
python llava/eval/eval_gpt4o.py \
    --api-key "your-api-key" \
    --question ./eval_data_json/eval_example.jsonl \
    --output ./eval_data_json/output_gpt4o_eval_example.jsonl \
    --max-tokens 1024
```
or use [`eval_gpt4o.sh`](https://github.com/Wangyixinxin/MMedAgent/blob/main/eval_gpt4o.sh)
### GPT-4 evalution
```
python ./llava/eval/eval_multimodal_chat_gpt_score.py \
    --question_input_path ./eval_data_json/eval_example.jsonl \
    --input_path ./eval_data_json/output_gpt4o_eval_example.jsonl
    --output_path ./eval_data_json/compare_gpt4o_medagent_reivew.jsonl
```
or use [`eval_gpt4.sh`](https://github.com/Wangyixinxin/MMedAgent/blob/main/eval_gpt4.sh)
## Data Download
### Instruction-tuning Dataset
We build the first open-source instruction tuning dataset for multi-modal medical agents.

| Data | size |
| --- | --- |
| xxx | xx MiB | 

### Tool dataset (Selected)

#### Grounding task dataset
| Data | size |
| --- | --- |
| xxx | xx MiB | 
#### Segmentation task dataset
| Data | size |
| --- | --- |
| xxx | xx MiB | 

## Model Download

## Web UI


## Citation
If you find this paper or code useful for your research, please cite our paper:
```
@article{li2024mmedagent,
  title={MMedAgent: Learning to Use Medical Tools with Multi-modal Agent},
  author={Li, Binxu and Yan, Tiankai and Pan, Yuanting and Xu, Zhe and Luo, Jie and Ji, Ruiyang and Liu, Shilong and Dong, Haoyu and Lin, Zihao and Wang, Yixin},
  journal={arXiv preprint arXiv:2407.02483},
  year={2024}
}
```
## Related Project
MMedAgent was built on [LLaVA-PLUS](https://llava-vl.github.io/llava-plus/) and [LLaVA-Med](https://github.com/microsoft/LLaVA-Med) was chosen as the backbone. 

## Contributing
We are working on extending the current tool lists to handle more medical tasks and modalities. We deeply appreciate any contribution made to improve the our Medical Agent. If you are developing better LLM-tools, feel free to contact us!
