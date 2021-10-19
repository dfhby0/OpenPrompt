

# 1. Workflow

## 1.1. config file reading

```python

from typing import OrderedDict
from yacs.config import CfgNode, _merge_a_into_b
import os
from collections import defaultdict

def get_config():
    parser = argparse.ArgumentParser("classification config")
    parser.add_argument("--config_yaml", type=str, help='the configuration file for this experiment.')
    parser.add_argument("--resume", action="store_true", help='whether to resume a training from the latest checkpoint.\
           It will fall back to run from initialization if no lastest checkpoint are found.')
    parser.add_argument("--test", action="store_true", help='whether to resume a training from the latest checkpoint.\
           It will fall back to run from initialization if no lastest checkpoint are found.') #
    args = parser.parse_args()
    config = get_yaml_config(args.config_yaml)
    check_config_conflicts(config)
    logger.info("CONFIGS:\n{}\n{}\n".format(config, "="*40))
    return config, args

def get_yaml_config(usr_config_path, default_config_path = "config_default.yaml"):
    """
    读取默认设置和模型设置，并merge
    依赖yacs.config 中CfgNode
    """
    # get default config
    cwd = os.path.dirname(__file__)
    default_config_path = os.path.join(cwd, default_config_path)
    config = get_config_from_file(default_config_path)

    # get user config
    usr_config = get_config_from_file(usr_config_path)

    config.merge_from_other_cfg(usr_config)

    config = get_conditional_config(config)  # 剔除无用的子设置
    return config

def get_config_from_file(path):
    cfg = CfgNode(new_allowed=True)
    cfg.merge_from_file(path)
    return cfg

def get_conditional_config(config):
    r"""Extract the config entries that do not have ``parent_config`` key.
    当设置中有某个父亲设置被赋予了这些子设置的key值，再将这些子设置放回config中（如learning_setting: few_shot）
    """
    deeper_config = CfgNode(new_allowed=True) # parent key to child node
    configkeys = list(config.keys())
    for key in configkeys:
        if config[key] is not None and 'parent_config' in config[key]:
            deeper_config[key] = config[key] # 取出含有parent_config的项
            config.pop(key)
    
    # breadth first search over all config nodes 
    queue = [config]
    
    while len(queue) > 0:
        v = queue.pop(0)
        ordv = OrderedDict(v.copy())
        while(len(ordv)>0):
            leaf = ordv.popitem()
            if isinstance(leaf[1], str) and \
                leaf[1] in deeper_config.keys():
                retrieved = deeper_config[leaf[1]]
                setattr(config, leaf[1], retrieved)
                if isinstance(retrieved, CfgNode): 
                    # also BFS the newly added CfgNode.
                    queue.append(retrieved)
            elif isinstance(leaf[1], CfgNode):
                queue.append(leaf[1])
    return config


config, args = get_config()
# init logger, create log dir and set log level, etc.

if not args.resume:
    EXP_PATH = config_experiment_dir(config) # 
else:
    EXP_PATH = config.logging.path  # 从已有的checkpoint进行训练/测试

init_logger(EXP_PATH+"/log.txt", config.logging.file_level, config.logging.console_level)
# save config to the logger directory
print(config.logging.path)



```