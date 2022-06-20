# bloom-demo
Early dev prototype for decentralized bloom. Not for public eyes **yet**.

```python
if you.read(this) and you.name not in '@timdettmers @borzunov @mryab @greenfatguy'.split():
  you.go("away")
```



# install


```bash
conda create -y --name bloom-demo python=3.8.12 pip
conda activate bloom-demo

conda install -y -c conda-forge cudatoolkit-dev==11.3.1 cudatoolkit==11.3.1 cudnn==8.2.1.32
pip install torch==1.11.0+cu113 torchvision==0.12.0+cu113 -f https://download.pytorch.org/whl/torch_stable.html
pip install accelerate==0.10.0 huggingface-hub==0.7.0
pip install bitsandbytes-cuda113==0.26.0
pip install https://github.com/learning-at-home/hivemind/archive/master.zip
pip install https://github.com/huggingface/transformers/archive/6589e510fa4e6c442059de2fab84752535de9b23.zip
```


### Test local inference:
No networking whatsoever, used to verify architecture optimizations

```bash
# run one bloom block for a few steps -- on a local machine
python -m cli.inference_one_block --config cli/config.json  # see other args
```

### Test distributed inference / training

First, run one or more servers like this:
```bash
# minimalistic server with non-trained bloom blocks
python -m cli.run_server --prefix bloom6b3 --converted_model_name_or_path bigscience/test-bloomd-6b3 \
  --block_indices 3:5 --torch_dtype float32 --identity_path ./server1.id --host_maddrs /ip4/127.0.0.1/tcp/31337
# when running multiple servers:
# - give each server a unique --identity_path (or remote --identity_path arg when debugging)
# - if running multiple servers on the same machine, give each a unique port (last integer in --host_maddrs, 0 means random port)
# - when running over the internet, change --host_maddrs according to https://learning-at-home.readthedocs.io/en/latest/user/dht.html#running-across-the-internet
# - each server except first should have --initial_peers pointing to one of pre-existing servers 
```

Then open a python notebook or console and run:
```python
import torch
import hivemind
from src.client.remote_block import get_remote_module

dht = hivemind.DHT(
    initial_peers=["/ip4/127.0.0.1/COPY_FULL_ADDRESS_FROM_ANY_OF_THE_SERVERS"],
    client_mode=True, start=True,
)

layer3, layer4 = get_remote_module(dht, ['bloom6b3.3', 'bloom6b3.4'])
assert layer3 is not None and layer4 is not None, "one or both layers were not found in DHT"
# test forward/backward, two blocks
outputs, = layer4(*layer3(torch.randn(1, 64, 4096)))
loss = (outputs * torch.randn_like(outputs)).norm()
loss.backward()

# test inference, one block
with layer3.begin_inference_session() as sess:
    for i in range(10):
        res = sess.step(torch.ones(1, 1, 4096))
```


### Convert regular bloom to distributed
```bash

# convert model from HF hub to a distributed format (can take hours depending on your connection!)
MY_WRITE_TOKEN=TODO_WRITE_TOKEN_FROM_https://huggingface.co/settings/token
python -m cli.convert_model --model bigscience/bloom-6b3  \
  --output_path ./converted_model --output_repo bigscience/test-bloomd-6b3 \
  --use_auth_token $MY_WRITE_TOKEN  # ^-- todo replace output repo with something you have access to
```