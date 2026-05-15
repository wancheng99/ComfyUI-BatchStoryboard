# ComfyUI-BatchStoryboard
ComfyUI custom node for AI movie storyboard batch saving with auto-naming and prompt metadata
import os
import json
from datetime import datetime

class BatchStoryboardSaver:
    
    def __init__(self):
        self.output_dir = ""
        self.type = "output"
        self.prefix_append = ""
    
    @classmethod
    def INPUT_TYPES(cls):
        return {
            "required": {
                "images": ("IMAGE",),                    # 输入图片批次
                "scene_number": ("INT", {               # 场次号
                    "default": 1, 
                    "min": 1, 
                    "max": 999
                }),
                "shot_number": ("INT", {                # 镜头号
                    "default": 1, 
                    "min": 1, 
                    "max": 999
                }),
                "prompt_text": ("STRING", {              # 该镜头的提示词
                    "default": "", 
                    "multiline": True
                }),
                "output_folder": ("STRING", {           # 输出目录名
                    "default": "storyboards"
                }),
            },
            "optional": {
                "project_name": ("STRING", {            # 项目名称前缀
                    "default": "AI_Movie"
                }),
            }
        }
    
    RETURN_TYPES = ()
    FUNCTION = "save_batch"
    OUTPUT_NODE = True
    CATEGORY = "AI电影/分镜工具"

    def save_batch(self, images, scene_number, shot_number, prompt_text, output_folder, project_name="AI_Movie"):
        # 获取ComfyUI标准输出路径
        base_output = os.path.join(os.getcwd(), "output", output_folder)
        os.makedirs(base_output, exist_ok=True)
        
        # 格式化命名：Project_Scene01_Shot03_时间戳
        timestamp = datetime.now().strftime("%m%d_%H%M%S")
        saved_files = []
        
        for idx, image in enumerate(images):
            # 文件名构造
            filename = f"{project_name}_Scene{scene_number:02d}_Shot{shot_number:02d}_{idx:03d}_{timestamp}.png"
            filepath = os.path.join(base_output, filename)
            
            # 保存图片（ComfyUI的IMAGE是tensor，需要转换）
            # 这里使用ComfyUI内置的save方法逻辑
            from comfy.utils import common_upscale
            import numpy as np
            from PIL import Image
            
            # tensor转PIL (ComfyUI IMAGE格式: B,H,W,C 范围0-1)
            img_np = (image.cpu().numpy() * 255).astype(np.uint8)
            img_pil = Image.fromarray(img_np)
            img_pil.save(filepath, "PNG")
            
            # 同步保存提示词txt
            txt_name = filename.replace(".png", ".txt")
            txt_path = os.path.join(base_output, txt_name)
            metadata = {
                "project": project_name,
                "scene": scene_number,
                "shot": shot_number,
                "frame_index": idx,
                "prompt": prompt_text,
                "saved_at": timestamp
            }
            with open(txt_path, "w", encoding="utf-8") as f:
                f.write(json.dumps(metadata, ensure_ascii=False, indent=2))
            
            saved_files.append(filename)
        
        print(f"[BatchStoryboard] 已保存 {len(saved_files)} 帧到 {base_output}")
        return {"ui": {"saved": saved_files}, "result": ()}

NODE_CLASS_MAPPINGS = {
    "BatchStoryboardSaver": BatchStoryboardSaver
}

NODE_DISPLAY_NAME_MAPPINGS = {
    "BatchStoryboardSaver": "分镜批量保存 🎬"
}
