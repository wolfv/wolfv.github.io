# Writeup of ipywidgets workshop

We've had a 3 days ipywidgets workshop in Paris, and while others worked on doing the hard work (fixing bugs and preparing the next big release of ipywidgets 8.0) I was finishing up an old PR for a new feature (drag and drop!) and working on JupyterLab dynamic extension loading support for the upcoming JupyterLab 2.0!

## Drag and Drop in ipywidgets

During summer, we've worked with [Bartosz Telenczuk](	https://github.com/btel) on drag'n'drop support in ipywidgets. This work went very far, and there was an open pull request with all the good ideas inside. The initial work was done, but some cleanup and last bits of work before this could be merged. This last bit of work is now done.

The initial implementation was a draggable Label that could be dropped on containers. 
After we started with an implementation of a draggable `Label`, we settled on more generic widgets: a 