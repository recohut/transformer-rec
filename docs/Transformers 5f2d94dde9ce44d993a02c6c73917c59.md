# Transformers

The old, obsolete, 1980 architecture of Recurrent Neural Networks(RNNs) including the LSTMs were simply not producing good results anymore. In less than two years, transformer models wiped RNNs off the map and even outperformed human baselines for many tasks.

Here’s what the transformer block looks like in PyTorch:

```python
class TransformerBlock(nn.Module):
  def __init__(self, k, heads):
    super().__init__()

    self.attention = SelfAttention(k, heads=heads)

    self.norm1 = nn.LayerNorm(k)
    self.norm2 = nn.LayerNorm(k)

    self.ff = nn.Sequential(
      nn.Linear(k, 4 * k),
      nn.ReLU(),
      nn.Linear(4 * k, k))

  def forward(self, x):
    attended = self.attention(x)
    x = self.norm1(attended + x)
    
    fedforward = self.ff(x)
    return self.norm2(fedforward + x)
```

Normalization and residual connections are standard tricks used to help deep neural networks train faster and more accurately. The layer normalization is applied over the embedding dimension only.

That is, the block applies, in sequence: a self attention layer, layer normalization, a feed forward layer (a single MLP applied independently to each vector), and another layer normalization. Residual connections are added around both, before the normalization. The order of the various components is not set in stone; the important thing is to combine self-attention with a local feedforward, and to add normalization and residual connections.

![Untitled](Transformers%205f2d94dde9ce44d993a02c6c73917c59/Untitled.png)

## The transformer block

1. Multi-head attention
2. Scaling the dot product
3. Queries, keys and values

The actual self-attention used in modern transformers relies on three additional tricks. 

### Additional tricks

```python
import torch
import torch.nn.functional as F

# assume we have some tensor x with size (b, t, k)
x = ...

raw_weights = torch.bmm(x, x.transpose(1, 2))
# - torch.bmm is a batched matrix multiplication. It 
#   applies matrix multiplication over batches of 
#   matrices.

weights = F.softmax(raw_weights, dim=2)

y = torch.bmm(weights, x)
```

We’ll represent the input, a sequence of t vectors of dimension k as a t by k matrix 𝐗. Including a minibatch dimension b, gives us an input tensor of size (b,t,k). The set of all raw dot products w′ij forms a matrix, which we can compute simply by multiplying 𝐗 by its transpose. Then, to turn the raw weights w′ij into positive values that sum to one, we apply a row-wise softmax. Finally, to compute the output sequence, we just multiply the weight matrix by 𝐗. This results in a batch of output matrices 𝐘 of size (b, t, k) whose rows are weighted sums over the rows of 𝐗. That’s all. Two matrix multiplications and one softmax gives us a basic self-attention.

### In Pytorch: basic self-attention

Let's understand with RecSys analogy. The tokens are both users and items. In movie recommenders e.g., to know a user's interest in different movies, we take a dot product of his embedding vector with movies' embedding vectors. Here in self-attention, we take a dot of the given token with all other tokens to know how much the given token is connected to other tokens.

[Transformers from scratch](http://peterbloem.nl/blog/transformers)

In effect, there are five processes we need to understand to implement this model:

- Embedding the inputs
- The Positional Encodings
- Creating Masks
- The Multi-Head Attention layer
- The Feed-Forward layer

### Embedding

Embedding is handled simply in PyTorch:

```python
class Embedder(nn.Module):
    def __init__(self, vocab_size, d_model):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, d_model)
    def forward(self, x):
        return self.embed(x)
```

When each word is fed into the network, this code will perform a look-up and retrieve its embedding vector. These vectors will then be learnt as a parameters by the model, adjusted with each iteration of gradient descent.

### Giving our words context: The Positional Encodings

In order for the model to make sense of a sentence, it needs to know two things about each word: what does the word mean? And what is its position in the sentence?

The embedding vector for each word will learn the meaning, so now we need to input something that tells the network about the word’s position.

*Vasmani et al* answered this problem by using these functions to create a constant of position-specific values:

$$PE_{(pos,2i)} = sin(pos/10000^{2i/d_{model}})$$

$$PE_{(pos,2i+1)} = cos(pos/10000^{2i/d_{model}})$$

[data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADwAAAAHCAIAAADbI7lLAAAC9klEQVR42mL5+/cvw1ADTAwMDB9ePl2/bsPRs9eePrh39/bdl+8/k24O4/69O5iYGBkYGP7/+3ny1DVMFXu2bGVi+Pvx68/P71/fuHX7/++fVy9dZWJiunLlyvc//x7cvf7rz++bV6+cPH7i+sl9dx49nzJvKSNM7/dv3/7+/sXAyHjixFkGBgbm+vr6t09v8isZVhWV/v/8yN4z6MK9e/Liooy4Hfj/378PHz7+/Pnj338mVlbmLx/f3Hn0kpFT4Mur+/cfv53d3xWXHH/67NlbT94KsDNcv/uUh/3fwxcf1+w6q6coeOPVn6L8ikB73aqGtoS0pNbOaa4ebkkpyRzCehPmb1Bm/f3n99OD5z78fn5VRkH1zpXrz5//ePPx7bnNK959/C8lL/3z/TMuURkWBgaGzRt3RWQU7Ny+Kruw6vqVW0bKUi/ffxbk4mBjZWZgZPz168+HF89nL15Tmhd58fprM3O93z+/TZs+nYWFydDcyd3R7O2zZ88+szAyME+d2GNmZf/k6f+fn18zcwvNaZuoben4/M6p4qSQ2y++ZKeF8XIxMH79/l/SkIuP58WbFwz//p/cfbawMPvjt68iMmL/7h516V3NwMDg/P8/AyMjw78/b99//PuHgZ2VUUIkgpWVlZmZSVhM5OnLd4x///5tb2wqra37/eVtS+/8Ty8fFxfFfHv3ZOX6A0pKekqy4hc+cOZGOW3af+LHjWM8cjpe3u6gVMXEBA/47r7JEeGht9+8/POD8/XV06zfXsnqaJ2//jo9I2ZS77T0vOTJk2dZm5mZWZi8e3Rr/61Pb6/ulRAVUVERf/KV09jQ8PShXb94VF89uKSib+tipoo/FZ7YtdXM3ZcRMyP++fWjc8bCsqyUn9+/sbKw/mNgevjirZQwDzMjw6Xbjy2NtFCSCjhQ/jMwMILY/xkZQcnq88cPPHwCYCZCkAGqCqYRVQuR4O/fv0xMTFgc/f8/AzMz09+/fzGNA0XafwbGgS49AAEAAP//NclyEQcXUK4AAAAASUVORK5CYII=](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADwAAAAHCAIAAADbI7lLAAAC9klEQVR42mL5+/cvw1ADTAwMDB9ePl2/bsPRs9eePrh39/bdl+8/k24O4/69O5iYGBkYGP7/+3ny1DVMFXu2bGVi+Pvx68/P71/fuHX7/++fVy9dZWJiunLlyvc//x7cvf7rz++bV6+cPH7i+sl9dx49nzJvKSNM7/dv3/7+/sXAyHjixFkGBgbm+vr6t09v8isZVhWV/v/8yN4z6MK9e/Liooy4Hfj/378PHz7+/Pnj338mVlbmLx/f3Hn0kpFT4Mur+/cfv53d3xWXHH/67NlbT94KsDNcv/uUh/3fwxcf1+w6q6coeOPVn6L8ikB73aqGtoS0pNbOaa4ebkkpyRzCehPmb1Bm/f3n99OD5z78fn5VRkH1zpXrz5//ePPx7bnNK959/C8lL/3z/TMuURkWBgaGzRt3RWQU7Ny+Kruw6vqVW0bKUi/ffxbk4mBjZWZgZPz168+HF89nL15Tmhd58fprM3O93z+/TZs+nYWFydDcyd3R7O2zZ88+szAyME+d2GNmZf/k6f+fn18zcwvNaZuoben4/M6p4qSQ2y++ZKeF8XIxMH79/l/SkIuP58WbFwz//p/cfbawMPvjt68iMmL/7h516V3NwMDg/P8/AyMjw78/b99//PuHgZ2VUUIkgpWVlZmZSVhM5OnLd4x///5tb2wqra37/eVtS+/8Ty8fFxfFfHv3ZOX6A0pKekqy4hc+cOZGOW3af+LHjWM8cjpe3u6gVMXEBA/47r7JEeGht9+8/POD8/XV06zfXsnqaJ2//jo9I2ZS77T0vOTJk2dZm5mZWZi8e3Rr/61Pb6/ulRAVUVERf/KV09jQ8PShXb94VF89uKSib+tipoo/FZ7YtdXM3ZcRMyP++fWjc8bCsqyUn9+/sbKw/mNgevjirZQwDzMjw6Xbjy2NtFCSCjhQ/jMwMILY/xkZQcnq88cPPHwCYCZCkAGqCqYRVQuR4O/fv0xMTFgc/f8/AzMz09+/fzGNA0XafwbGgS49AAEAAP//NclyEQcXUK4AAAAASUVORK5CYII=)

[data:image/png;base64,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](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAQYAAAAgCAIAAAB/1UH0AAARxElEQVR42ux8AWwTV5r/t/9a/wkX7ZppIyeXXOOjiEFEOG1vSVQQY4FqBAijFFL3coctfJd2UVOSDeye2k12aUl2082mmyYXGiC3uU2EV3BODeti2roMwq1RGxIQJmPVxSlBhgOTOTMX5y5u3smWTu+N7diOndBCdnOtP0Vk5r15b9587/t93/fe7xFZJBKBrGQlKzH5f1kVZCUrWUhk5U8u97y2kxbLGaf3XmIpCgrB+LXzJOdDCZVhxF9y2sym4TtZSGTl2yXIY8lbuc50ZmDP7h3rVm503omXm5aXLOclGAjOHT/61WRiMxlFjZ811vTmyLOQyMq3CxG2g4f6RgInek8EPj+lAf6dc16pgnpi+4n+U8WP4Guv43dgaFJRSS2XwiSsrmZy/+yQQAjCkPSTlUyz/TWV473g9AYXcDxeh83i8C4uHYWDir192kJyrWB1GhifRpKZoWmqtKxULgOAIFfDNewujTcSRnnew3900qSpLqcevpacArp/SNzhNhYV5RXkJf1sMHaf4RMxg1IwQ34QQt8pPPDHjEUFeSb3fdp40LIvb91zFiRbOOPztjxv3HOET5wGdId3phth8AZvM1ssJy3DHmG2EfMOm8VssdiH05hO0Oc8Y7GYLdwl7+xKn8uJG57kvPGWMgW7Xhk39Wsc6NcyxC57i5YvLzHfJMXDB0CrXU0yJORr35BXZx8fP7Ox3gq6Z5QP35EJvSVFVU4hQ3VktiCPgcbS/L4/Eon4L5rV5LZ2YIzUjrWxdEYxmkOR74rYG4iWzvnv6+GDWIWehdRO6HJP0ngEj/kgnkn1EVfKk67jjTRN61rN5iO1+KLTMVMnunARrTO/Z5YuHLcSKq+aSVmbdaCLXHT50UyluQ6XtR239pCLro9TNeN/v5Y2WhPNrN+NNeI5rqPr7NIjuN/OQfw5V3uwxtDXUoHf2tlm/SJJy6EvHW0NXSma9xw30LTONZmmjzSQCH1BPps2uGK9uI4YUszd1UdK2LaxUCSCIqFQSLzl6tLRavIx3xURPI5zLvE+5sx/rpmmabN7Yd2FA6MuNs1fmmlaV2vE3szQ50l8TPy4Dc9dgz06lZ06mqYb3xubsUiatkswEAdxHd04Jn2j4CDOsVm6FS/2JPbjaFXH3WgkIvaQltYvE158y07TzXEIEYs3EIsXu2i67bIYiUTGBmrjMPAcUcdw8jXE1anrupqiZ7GL1blSdS/2sBj4ofuBhKcvFQCe41HHESsJ9ZMPNhxP0rXniNrwnj+SlVQf4zIkmM5CCfLUzo7SGBgpkBC72ASjj0Qiky4ymdhYxYtddPJQXUcIYIihD3bOXM9YFU3b/Rg86lgniSGLbojFH3HQEIcWComTkcFWOmrxfjtN64gXD5lZWt1JYhryN+OIh3Hiv+qwvu+w97W19TnG3I6uhmarG5eHbrnMR3p6+qxjxAv4L1v7j5sbaboHu57Q4Hvmrs5+l18yV8MsSERCbqycto/FlPLZy+vg0BkbAOh3rokua8LCuzUmXPJCbD9gymvj8O/tZUy0zR2cOSq3HT24QfHtW0IPm1vyiGysqqraZ0EA/Mne9rda6qvy8qrwrddh6ZZu99kAEHeoHj+9oVsgi28fN2ADaP1bdmbteMFUX22s2pC3cV+v094rdZ5X1Z608p7y2Q4dyIvVmS4Jc4wHl7qdJoD9Bpaae/UvjBxwA0DNivhE5RZrNQDQ7rwFIx8O4LpnVsQfZ9ZqAaD7wyEIC8438KyXM/GW8vJqXGu74BOunMVrTUOpIrZSolaU47rDFm8YYIqvWr61uH87+oLn3cPd+t3DU+iaDfRrg72HOLfjd2B4kZHGXQBAUQDIVFDSDqqVwPWe8cKE07iLL91Zrejd8bonv7q23Mi+i8Le3aWn2Zeq9WsmfqjsDt6wlPTn6F+o3PSyCiLAH9ryB1BpCrmNJd2ZFrjUKrYBoOUQh+ZZXgej5s4+sRTCSPA42/Ul7bigpmZbDAAeJ35kdQO7TFI8/1xpnS8MVCGjzF2MRu011+fNIy2+THtHd85urWmHl08FAoGmNePcla8AoPgZVjk9YuJAuxO7CWqpAm6fNXGwn83p3lBUdUWhXw3gHhqPYO1c4roBNKWPz9iqomx79TaGcwN/7NUdv0fnPzvfWqECruWAObqHgUZtG5U/NL4B568FAoHrrRVc/ZYS7k7G8QDAkM0EoNn09DwuCQUJtCqeLJhZ5ecsIbN29fNR4QYewJNMQYLhLMH/HhvxhSdG8JWeeTyhkloKAKYroxMC7la7lkmtA5NXCJqqN3IA3bu3rtuwceOGrQf+8kWNApYUgKnmNPNC2Sc13P5dZVIbxXoN/7N1eXm7UXcTAL/n2FfPb2bkj+Vr3tQocuXlu7TsKoZ6tFi7GtCok1tdLMfYY7Uw4Dx3uqaiFADyi5UI0NV3+TX5AKsaLo/syrxhpWAMANyhoST2EFK3P4KjwwQRsIddvidapml4+5Xqv2PlsWdHHNiXgPv00IUy+ZRw9s09fP0pRrZ4/TyVq1BptHPA1TcFKDJbGZINTeBf/40dOPv3e1VBBcZAIaNAWE/bn1YCgPIplrHgp9p/VKV983zgJZWlut3kBjIZwfEr2OzkOYkDklNhyevrr5+okQMw+/SvWl/lznnRSyrq3vDutUYetB9cb1LJiTN+WgVW/m4QQSGVdjwQ9lo6eKjYWzofqzUdHCchKEk9qs16sJpuiv9VcB3ff5XgHahlKj2ACbyT/zk+jQsmIEFRyqdZXDkaDOTfJcpKtCwlWwEmK0wEc/QnAvpZI6nsv135CAUyKPfdpnKjdsvuPXH7HxBQFCUD/bbqaLmMAmkzMydmswUUpVCCe3gagJKhaWALc73dl8abokkKtaQAxmX5zCq5cIHzPqbBsH4kjTaYtXo4ZvIJiH2MyggJKXRC/Ynbr2kggiAM8eHGQ6/z19iXqDRPDtktXqeJc0PTW6WzbYm3DxU/OwOkBxEcl3iBUjDsetVMh+GgMJUzcWVo6VpWMefetXJbw/lt3xROxU9qALhjxvYtn+/fXPnhz6PbnWcP48i5Zhkl3X5ymDh4zdG+l1QwNTxgBajYrsQpwMSQOw3Qhk7iXLRvuEOy4WmyVa9ZX0xBsLdyKwdQ03+wLGbf05APwM8xHvQFyZpeYOfdwh/3DKXDPYZZ8fcn0ww1goixMzn3BC7Nti+JUSv+wn8lTeVXBHhL5RkGRVExh0Ul+6/Uci/v5d4QvC9Q3Bmbl+J9yGvjuJHI0fNvW378lmW7/NqmP+79m7KJmqJ1xmCN77At/zVnz+tHn2CXHwDY3//pT+44WzgbO+hTrb+v/dxkgw0Lzg6s+oYNZZSMoHP259wZIXmU6jdHO8ictXbn7S5bleKdkKW6yLnt0w7ZQ0l7jOtqbFoNkIyu4fO7+6WE1fv75et+1jQ2zFQU1R+93cHMaQ7zciYUlaF9rqrxbS23z9ayq4QZCUiUE7rBdwPAy2qlTMpzyC3Aqd5KAmAcafXbyGIsnXOKLsY0HZuWxckj7IlyFHIIes9iu9TrNfH5CzrfxV++NIfKNJ6hP5Ks6Zn5F3LKNdsBbOn1/B/f11WAzZpaTlyzd7r4eW2aliStGr23cieOM6l1JCyPizi4PYgBMDubAjvJxYkAKVAFAljPYOjoCyMEldhWQdEUCCAEVHOT1CpwtxJFJNwxfYHAN2Wvbw0Rc9ewqowB2HflE/KInpEeCd/9wRsvlibnJN5ju/cUnurYyTwc+qmm+PLdQN+JwOcftgK0fDSKYr7/g753NT9YpvmQK1632TKHyQv2A0XzyBY+XXth1IcAVIajfS/j29/+qzOqhMHTALD/2Whs9F7Ct9ruT9ncmcRSI3FMMuX2CnI04ZHE7HSIA9DujPGyiH8HB17NK5sZdOcaNn+DJo5wNMq1uAFWt7LLMownmjXpSu//LFBucjx3YFMvLU6DKHTTS2DA5KfDto8nA1it/P8wMWvifKcJupSPUguYE8soSjY78ES9PfWN3ixLNncS/jTazMpFl05ih6jdxspjKaN+r1I62vVbj6phJwOCc92+nE9vs7FcPBgUJ6jCAnTTOx7OZ1bN5DjBG96bQchfwSiiM4R8bu8klZ//qFwRz+2C6JXPGiVnrHiK1cbyVTSFpill+dN4FNRT+/uW5/3KzjZtzuAmc5n9rzUszcnJlGBPTCvSTTk6vdbI+M6zuZT2H4/C4T2824eApaI2pNmkAu4YxxrKyB6dqvpZJhppf83D6tbyXL79l+Ov/FyjwI7cdFPoYAqj/d68cBarWRWNA86uOhuAvrupTA5IWJK8Tg0O/AKv6Y7+i14OqDfdeIBkTTX3kTWRvIvBeZfV6TtaGVv+TY+TJUS56vH8m1qw2jjep1/FJK09Xi5X5Co1OFDb+JuIWRF9VVDAWVVN2UplkQbAZnPwyBDD8lSQrD32ly7iPUjvZzi6rkwJYgkb236JltZ1uubY/JaIbXMyQRi55dDRdBdhWzwDBpqN746L1oMSra1u62zGnfeRzkOeZpbWddpd57pomnaQnW57Ha3utDuO19J0f3pOS3BEqSgk2jvxQKQ3RunPOC360Hb6xxrj3IvfoabjW+aeRpqm2drmOppuHYzyWXHSJkRUxBoMsS18abO/bYbKlZgBgwtJpGcyf0yYBLpO6i0U47/G5hiP4yC+HBTTfYLESyTRRyGzEb+hP84b+h1kgrrEOEubQG6QzqOb9xI9lcBy+JvJzDrEGCtC18a3/yV2Ut26mKlbMv5ZbB0kET1xqUt/LsP/frNk3+bL/tBkSBRFz1VHf6vUNErTYOucgUQkdLUf65QQilYjTbP9IUQo0lbJCMQ2mm67iKejn6XVRzyRiGge8KQ/E1FHG6RDJZFIBE+eIc7Sk5f2hx42xaaTtKEjv41dY9EX+Nuk8gZz3Ixq4wOLeY3G+FcgSfU9YiSRA6ZpYy3pV9f/8VgS/9rXKL2VPFZrd4tzjQeN1dLpWdgZjjWFv7tlJycPuvwSQsjhi/6rYgIFHj2LIZFZtK5fjPmIZumIhz+SgBBXom1EsT0pnQoxpD0xsVgYVEImdl0U52evM1Ll0rGODBJ3Hta6pDkYbI1TmCGzDrtV//uNM+cOyDT3EMB4BmqTvWkKGhuTWVV1oh0Q4Bke+gkikQxSFERRTO46NKskqTYUSnlc8gvk0/zn2iRLCk1m7gSF0rw0w3hCgl+cTBPimpPmR5d48ke8Kp1bU0sY60nSuWhuwJVqowS8Hn/iiRXR1ZyASsMRRxICB0glK/VqSDwctehE8lPpjhTAQ39XssP2x4KAlPbQPZdF7IdiSY54sQ3DQ4yIYsxjsT2zUwDxYo+6LpYXiWIIiV00XZtweGRBosRDFXIqTD0oRFOR1Mzzz2ATov9Lv/+WPy0qRT+u8vvF9O7pFqnN1JLUhtCiPmRDgmHsgMl8BzoeVPILS6N7cwBw71oLqDap5ADB3sodUH+q+ik5ugdAGC4Ie3+8paXmD/+syvW+vscEQFXWdIDbNxEGCPIt1UaLKxj9v1dbXt27m+HdPO+yGSveCaLxTwDWgLP3TOw/owBAwZLFfCxEZej74E3l1pXPvEO2uRULug9zXxsrcsUyhaJQkZY2kCtwlUKRfpsFVxUqMrYktdQipm6dv8zb0VHzqa9JKbvPw+EPfP5MJx0Fi62x2o6bm3V0bd9g0nGrvp5aVt11biyePtV29rQZ1bXHXTOncXXm0KQrJV1rfG9MegWti8d0vA6Zdf5xEQZrzz/RNL1erWZTjlVn5U+7rHbPlWJ/byH+aI3gaCl5Y+VtR6XzF3lV8lPXa8tzHknaP4YwCk5Ny3PlM5vAYQAURDJ5fC85eKH9Odem83tVmai3OLkmnKkv4bSBtzWQlaw8ePhciE4VGxrO76ra8pv/efwwNHCl8tmUiYySp4RdGQ7lVALR02tX/NvrqnnJZuFCd0kbc92RxUNWHo58b+H+tNlHb/3U9O/hv/4rzU9+ql24v7EgjHrlKxgqO5NZWfyQyEpW/i9K9o/WZCUrWUhkJStZSGQlK1lIZCUrWUhkJSsPLP8bAAD//w2N0OEkXq0VAAAAAElFTkSuQmCC)

[data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADwAAAAGCAIAAAAQf2ruAAACz0lEQVR42tSTy28bVRTG7523PS97PPbYTuq6itu0tI3dgEPVKpUQUCEQKCy6AFb8I2xBsIANqypCLLqpEBJCQkg8pFS80tbCqp3GqE2dNn5lbGecjD3vOxdZWRAWICRWPbvz09F3jj59h0IIgaetKABAiIJ6tUZF+aQijS1bFBVVlf+/9GjYE2MpkiSOwseNjXg2x7Cc71l+SNAAHdieSBEugoLATvxQf/LHRgvNxqGmaY0H91975VWMQRgijEEQIMMYKmoSIoR8y3jv/U9SJJlOC/mFKxSvlYvH/stZxqDX0YcEyZ0+NWebo1tr3xcXL1Xu3D57stBoDXb6RvF4rN1rX77y8p1ffi4/t3D33uPt/mBpPsPKM9/e/CKSEd2AWTzG19vjNB/ZbOknJD95sqRwcKfvRCjLhrFsIipJihOEELtdFH/hlPTNrdrU6Yf366XLL129WPzogw+FgrV8hjOMIcVEaBhCkoEErP16m9ZmcukUdsyoxA9Mz7WduXx20G7+9Ns9lpPn5ws3Vldff+fdrz67HtdizUfuYCQqKl+pbs5qyurHn545n282m4+6/fKziykxHFn+xOM4Z2K4UJBEUg8EKmz1RqUZeaF4QZGlCwyNXBcFfqfX8UOWoV3oh/kEL8rxdmtr6vQPX3+pnr14Lqde//xmPin5XFy0t+tDau54inaCwrnzIbLHQ52VE/W171689vZ+yHW36peeX/JcFyEMIGAYtrq+NgYi5+3z6RNPqnfzxfLvlfUHW9srKyvI3KXiWWOnRvOx0tJyc6MyBoLVbLiyyuHAngwQSXuElBGorq6/8ea1vwUKAwD/6vTWw82OA48+IkFMxw29e+PH9beuLtsTiyXAaG/PxWQkKuTyswe7u6KaMM1JVzefOZ379/D4ntMd7ucy2lGIMT7cgjE+JBBCjAGEUwIBCDGGEP6T5nhsCrz4ZwAAAP//3alPdEX0iW0AAAAASUVORK5CYII=](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADwAAAAGCAIAAAAQf2ruAAACz0lEQVR42tSTy28bVRTG7523PS97PPbYTuq6itu0tI3dgEPVKpUQUCEQKCy6AFb8I2xBsIANqypCLLqpEBJCQkg8pFS80tbCqp3GqE2dNn5lbGecjD3vOxdZWRAWICRWPbvz09F3jj59h0IIgaetKABAiIJ6tUZF+aQijS1bFBVVlf+/9GjYE2MpkiSOwseNjXg2x7Cc71l+SNAAHdieSBEugoLATvxQf/LHRgvNxqGmaY0H91975VWMQRgijEEQIMMYKmoSIoR8y3jv/U9SJJlOC/mFKxSvlYvH/stZxqDX0YcEyZ0+NWebo1tr3xcXL1Xu3D57stBoDXb6RvF4rN1rX77y8p1ffi4/t3D33uPt/mBpPsPKM9/e/CKSEd2AWTzG19vjNB/ZbOknJD95sqRwcKfvRCjLhrFsIipJihOEELtdFH/hlPTNrdrU6Yf366XLL129WPzogw+FgrV8hjOMIcVEaBhCkoEErP16m9ZmcukUdsyoxA9Mz7WduXx20G7+9Ns9lpPn5ws3Vldff+fdrz67HtdizUfuYCQqKl+pbs5qyurHn545n282m4+6/fKziykxHFn+xOM4Z2K4UJBEUg8EKmz1RqUZeaF4QZGlCwyNXBcFfqfX8UOWoV3oh/kEL8rxdmtr6vQPX3+pnr14Lqde//xmPin5XFy0t+tDau54inaCwrnzIbLHQ52VE/W171689vZ+yHW36peeX/JcFyEMIGAYtrq+NgYi5+3z6RNPqnfzxfLvlfUHW9srKyvI3KXiWWOnRvOx0tJyc6MyBoLVbLiyyuHAngwQSXuElBGorq6/8ea1vwUKAwD/6vTWw82OA48+IkFMxw29e+PH9beuLtsTiyXAaG/PxWQkKuTyswe7u6KaMM1JVzefOZ379/D4ntMd7ucy2lGIMT7cgjE+JBBCjAGEUwIBCDGGEP6T5nhsCrz4ZwAAAP//3alPdEX0iW0AAAAASUVORK5CYII=)

[data:image/png;base64,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](data:image/png;base64,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)

This constant is a 2d matrix. $*pos*$ refers to the order in the sentence, and *i* $$refers to the position along the embedding vector dimension. Each value in the $pos/i$ matrix is then worked out using the equations above.

![Untitled](Transformers%205f2d94dde9ce44d993a02c6c73917c59/Untitled%201.png)

The positional encoding matrix is a constant whose values are defined by the above equations. When added to the embedding matrix, each word embedding is altered in a way specific to its position.

An intuitive way of coding our Positional Encoder looks like this:

```python
class PositionalEncoder(nn.Module):
    def __init__(self, d_model, max_seq_len = 80):
        super().__init__()
        self.d_model = d_model
        
        # create constant 'pe' matrix with values dependant on 
        # pos and i
        pe = torch.zeros(max_seq_len, d_model)
        for pos in range(max_seq_len):
            for i in range(0, d_model, 2):
                pe[pos, i] = \
                math.sin(pos / (10000 ** ((2 * i)/d_model)))
                pe[pos, i + 1] = \
                math.cos(pos / (10000 ** ((2 * (i + 1))/d_model)))
                
        pe = pe.unsqueeze(0)
        self.register_buffer('pe', pe)
 
    
    def forward(self, x):
        # make embeddings relatively larger
        x = x * math.sqrt(self.d_model)
        #add constant to embedding
        seq_len = x.size(1)
        x = x + Variable(self.pe[:,:seq_len], \
        requires_grad=False).cuda()
        return x
```

The above module lets us add the positional encoding to the embedding vector, providing information about structure to the model.

### **Creating Our Masks**

Masking plays an important role in the transformer. It serves two purposes:

- In the encoder and decoder: To zero attention outputs wherever there is just padding in the input sentences.
- In the decoder: To prevent the decoder ‘peaking’ ahead at the rest of the translated sentence when predicting the next word.

Creating the mask for the input is simple:

```python
batch = next(iter(train_iter))
input_seq = batch.English.transpose(0,1)
input_pad = EN_TEXT.vocab.stoi['<pad>']# creates mask with 0s wherever there is padding in the input
input_msk = (input_seq != input_pad).unsqueeze(1)
```

For the target_seq we do the same, but then create an additional step:

```python
# create mask as beforetarget_seq = batch.French.transpose(0,1)
target_pad = FR_TEXT.vocab.stoi['<pad>']
target_msk = (target_seq != target_pad).unsqueeze(1)size = target_seq.size(1) # get seq_len for matrixnopeak_mask= np.triu(np.ones(1, size, size),
k=1).astype('uint8')
nopeak_mask = Variable(torch.from_numpy(nopeak_mask)== 0)target_msk = target_msk & nopeak_mask
```

### **Multi-Headed Attention**

Once we have our embedded values (with positional encodings) and our masks, we can start building the layers of our model.

Here is an overview of the multi-headed attention layer:

![Untitled](Transformers%205f2d94dde9ce44d993a02c6c73917c59/Untitled%202.png)

Multi-headed attention layer, each input is split into multiple heads which allows the network to simultaneously attend to different subsections of each embedding.

A scaled dot-product attention mechanism is very similar to a self-attention (dot-product) mechanism except it uses a scaling factor. The multi-head part, on the other hand, ensures the model is capable of looking at various aspects of input at all levels. Transformer models attend to encoder annotations and the hidden values from past layers. The architecture of the Transformer model does not have a recurrent step-by-step flow; instead, it uses positional encoding in order to have information about the position of each token in the input sequence. The concatenated values of the embeddings (randomly initialized) and the fixed values of positional encoding are the input fed into the layers in the first encoder part and are propagated through the architecture.

In the case of the Encoder, *V, K* and *G* will simply be identical copies of the embedding vector (plus positional encoding). They will have the dimensions Batch_size * seq_len * d_model.

In multi-head attention we split the embedding vector into *N* heads, so they will then have the dimensions $batch\_size * N * seq\_len * (d\_model / N).$

This final dimension $(d\_model / N )$ we will refer to as $d\_k$.

Let’s see the code for the decoder module:

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, heads, d_model, dropout = 0.1):
        super().__init__()
        
        self.d_model = d_model
        self.d_k = d_model // heads
        self.h = heads
        
        self.q_linear = nn.Linear(d_model, d_model)
        self.v_linear = nn.Linear(d_model, d_model)
        self.k_linear = nn.Linear(d_model, d_model)
        self.dropout = nn.Dropout(dropout)
        self.out = nn.Linear(d_model, d_model)
    
    def forward(self, q, k, v, mask=None):
        
        bs = q.size(0)
        
        # perform linear operation and split into h heads
        
        k = self.k_linear(k).view(bs, -1, self.h, self.d_k)
        q = self.q_linear(q).view(bs, -1, self.h, self.d_k)
        v = self.v_linear(v).view(bs, -1, self.h, self.d_k)
        
        # transpose to get dimensions bs * h * sl * d_model
       
        k = k.transpose(1,2)
        q = q.transpose(1,2)
        v = v.transpose(1,2)
				# calculate attention using function we will define next
        scores = attention(q, k, v, self.d_k, mask, self.dropout)
        
        # concatenate heads and put through final linear layer
        concat = scores.transpose(1,2).contiguous()\
        .view(bs, -1, self.d_model)
        
        output = self.out(concat)
    
        return output
```

### Calculating Attention

![Untitled](Transformers%205f2d94dde9ce44d993a02c6c73917c59/Untitled%203.png)

$$Attention(Q,K,V) = \mathrm{softmax}\left(\frac{\mathbf Q \mathbf K^\top }{\sqrt{d}}\right) \mathbf V \in \mathbb{R}^{n\times v}$$

Initially we must multiply Q by the transpose of K. This is then ‘scaled’ by dividing the output by the square root of $d\_k$. Before we perform Softmax, we apply our mask and hence reduce values where the input is padding (or in the decoder, also where the input is ahead of the current word). Finally, the last step is doing a dot product between the result so far and V.

Here is the code for the attention function:

```python
def attention(q, k, v, d_k, mask=None, dropout=None):
    scores = torch.matmul(q, k.transpose(-2, -1)) /  math.sqrt(d_k)

		if mask is not None:
        mask = mask.unsqueeze(1)
        scores = scores.masked_fill(mask == 0, -1e9)

		scores = F.softmax(scores, dim=-1)
    
    if dropout is not None:
        scores = dropout(scores)
        
    output = torch.matmul(scores, v)
    return output
```

In PyTorch, it looks like this:

```python
from torch import Tensor
import torch.nn.functional as f

def scaled_dot_product_attention(query: Tensor, key: Tensor, value: Tensor) -> Tensor:
    temp = query.bmm(key.transpose(1, 2))
    scale = query.size(-1) ** 0.5
    softmax = f.softmax(temp / scale, dim=-1)
    return softmax.bmm(value)
```

Note that MatMul operations are translated to `torch.bmm` in PyTorch. That’s because Q, K, and V (query, key, and value arrays) are batches of matrices, each with shape `(batch_size, sequence_length, num_features)`. Batch matrix multiplication is only performed over the last two dimensions.

The attention head will then become:

```python
import torch
from torch import nn

class AttentionHead(nn.Module):
    def __init__(self, dim_in: int, dim_k: int, dim_v: int):
        super().__init__()
        self.q = nn.Linear(dim_in, dim_k)
        self.k = nn.Linear(dim_in, dim_k)
        self.v = nn.Linear(dim_in, dim_v)

    def forward(self, query: Tensor, key: Tensor, value: Tensor) -> Tensor:
        return scaled_dot_product_attention(self.q(query), self.k(key), self.v(value))
```

Now, it’s very easy to build the multi-head attention layer. Just combine `num_heads` different attention heads and a Linear layer for the output.

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, num_heads: int, dim_in: int, dim_k: int, dim_v: int):
        super().__init__()
        self.heads = nn.ModuleList(
            [AttentionHead(dim_in, dim_k, dim_v) for _ in range(num_heads)]
        )
        self.linear = nn.Linear(num_heads * dim_v, dim_in)

    def forward(self, query: Tensor, key: Tensor, value: Tensor) -> Tensor:
        return self.linear(
            torch.cat([h(query, key, value) for h in self.heads], dim=-1)
        )
```

Let’s pause again to examine what’s going on in the `MultiHeadAttention` layer. Each attention head computes its own query, key, and value arrays, and then applies scaled dot-product attention. Conceptually, this means each head can attend to a different part of the input sequence, independent of the others. Increasing the number of attention heads allows us to “pay attention” to more parts of the sequence at once, which makes the model more powerful.

### **The Feed-Forward Network**

This layer just consists of two linear operations, with a relu and dropout operation in between them.

```python
class FeedForward(nn.Module):
    def __init__(self, d_model, d_ff=2048, dropout = 0.1):
        super().__init__() 
        # We set d_ff as a default to 2048
        self.linear_1 = nn.Linear(d_model, d_ff)
        self.dropout = nn.Dropout(dropout)
        self.linear_2 = nn.Linear(d_ff, d_model)
    def forward(self, x):
        x = self.dropout(F.relu(self.linear_1(x)))
        x = self.linear_2(x)
        return x
```

The feed-forward layer simply deepens our network, employing linear layers to analyze patterns in the attention layers output.

### **One Last Thing : Normalization**

Normalisation is highly important in deep neural networks. It prevents the range of values in the layers changing too much, meaning the model trains faster and has better ability to generalise.

![Untitled](Transformers%205f2d94dde9ce44d993a02c6c73917c59/Untitled%204.png)

We will be normalizing our results between each layer in the encoder/decoder, so before building our model let’s define that function:

```python
class Norm(nn.Module):
    def __init__(self, d_model, eps = 1e-6):
        super().__init__()
    
        self.size = d_model
        # create two learnable parameters to calibrate normalisation
        self.alpha = nn.Parameter(torch.ones(self.size))
        self.bias = nn.Parameter(torch.zeros(self.size))
        self.eps = eps
    def forward(self, x):
        norm = self.alpha * (x - x.mean(dim=-1, keepdim=True)) \
        / (x.std(dim=-1, keepdim=True) + self.eps) + self.bias
        return norm
```

### Putting it all together!

Let’s have another look at the over-all architecture and start building:

![Untitled](Transformers%205f2d94dde9ce44d993a02c6c73917c59/Untitled%205.png)

**One last Variable:** If you look at the diagram closely you can see a ‘Nx’ next to the encoder and decoder architectures. In reality, the encoder and decoder in the diagram above represent one layer of an encoder and one of the decoder. N is the variable for the number of layers there will be. Eg. if N=6, the data goes through six encoder layers (with the architecture seen above), then these outputs are passed to the decoder which also consists of six repeating decoder layers.

We will now build EncoderLayer and DecoderLayer modules with the architecture shown in the model above. Then when we build the encoder and decoder we can define how many of these layers to have.

```python
# build an encoder layer with one multi-head attention layer and one # feed-forward layer
class EncoderLayer(nn.Module):
    def __init__(self, d_model, heads, dropout = 0.1):
        super().__init__()
        self.norm_1 = Norm(d_model)
        self.norm_2 = Norm(d_model)
        self.attn = MultiHeadAttention(heads, d_model)
        self.ff = FeedForward(d_model)
        self.dropout_1 = nn.Dropout(dropout)
        self.dropout_2 = nn.Dropout(dropout)
        
    def forward(self, x, mask):
        x2 = self.norm_1(x)
        x = x + self.dropout_1(self.attn(x2,x2,x2,mask))
        x2 = self.norm_2(x)
        x = x + self.dropout_2(self.ff(x2))
        return x
    
# build a decoder layer with two multi-head attention layers and
# one feed-forward layer
class DecoderLayer(nn.Module):
    def __init__(self, d_model, heads, dropout=0.1):
        super().__init__()
        self.norm_1 = Norm(d_model)
        self.norm_2 = Norm(d_model)
        self.norm_3 = Norm(d_model)
        
        self.dropout_1 = nn.Dropout(dropout)
        self.dropout_2 = nn.Dropout(dropout)
        self.dropout_3 = nn.Dropout(dropout)
        
        self.attn_1 = MultiHeadAttention(heads, d_model)
        self.attn_2 = MultiHeadAttention(heads, d_model)
        self.ff = FeedForward(d_model).cuda()

		def forward(self, x, e_outputs, src_mask, trg_mask):
        x2 = self.norm_1(x)
        x = x + self.dropout_1(self.attn_1(x2, x2, x2, trg_mask))
        x2 = self.norm_2(x)
        x = x + self.dropout_2(self.attn_2(x2, e_outputs, e_outputs,
        src_mask))
        x2 = self.norm_3(x)
        x = x + self.dropout_3(self.ff(x2))
        return x

		# We can then build a convenient cloning function that can generate multiple layers:
		def get_clones(module, N):
		    return nn.ModuleList([copy.deepcopy(module) for i in range(N)])
```

We’re now ready to build the encoder and decoder:

```python
class Encoder(nn.Module):
    def __init__(self, vocab_size, d_model, N, heads):
        super().__init__()
        self.N = N
        self.embed = Embedder(vocab_size, d_model)
        self.pe = PositionalEncoder(d_model)
        self.layers = get_clones(EncoderLayer(d_model, heads), N)
        self.norm = Norm(d_model)
    def forward(self, src, mask):
        x = self.embed(src)
        x = self.pe(x)
        for i in range(N):
            x = self.layers[i](x, mask)
        return self.norm(x)
    
class Decoder(nn.Module):
    def __init__(self, vocab_size, d_model, N, heads):
        super().__init__()
        self.N = N
        self.embed = Embedder(vocab_size, d_model)
        self.pe = PositionalEncoder(d_model)
        self.layers = get_clones(DecoderLayer(d_model, heads), N)
        self.norm = Norm(d_model)
    def forward(self, trg, e_outputs, src_mask, trg_mask):
        x = self.embed(trg)
        x = self.pe(x)
        for i in range(self.N):
            x = self.layers[i](x, e_outputs, src_mask, trg_mask)
        return self.norm(x)
```

And finally… The transformer!

```python
class Transformer(nn.Module):
    def __init__(self, src_vocab, trg_vocab, d_model, N, heads):
        super().__init__()
        self.encoder = Encoder(src_vocab, d_model, N, heads)
        self.decoder = Decoder(trg_vocab, d_model, N, heads)
        self.out = nn.Linear(d_model, trg_vocab)
    def forward(self, src, trg, src_mask, trg_mask):
        e_outputs = self.encoder(src, src_mask)
        d_output = self.decoder(trg, e_outputs, src_mask, trg_mask)
        output = self.out(d_output)
        return output
# we don't perform softmax on the output as this will be handled 
# automatically by our loss function
```