# nClothPerVertexEdit
Maya Python plugin for setting and getting per vertex attributes on nCloth or nRigid objects

# Plugin Contents
## Commands
* setNClothPerVertex
* getNClothPerVertex

# Example Usage
Simply load the plugin from your plugin manager and choose the nClothPerVertexEdit.py file.
## setNClothPerVertex
Set the newly created cubes perVertex attributes with the given values
```
from maya import cmds
cmds.polyCube()
cmds.nClothCreate()
# NOTE: The given list lenght must match the amount of vertices a mesh has if this is not the case might cause undefined behaviour
cmds.setNClothPerVertex(plugName = "inputAttractPerVertex", vertexWeight = [0.8, 0.1, 0.5, 0.1, 0.5, 1.0, 1.0, 0.1])
```
## getNClothPerVertex
If the current selection is a valid nCloth or nRigid object retrieves the given map for it.
Note in this example a nRigid does not have a inputAttractVertex map
```
from maya import cmds
cmds.polyCube()
cmds.nClothCreate()
cmds.setNClothPerVertex(plugName = "inputAttractPerVertex", vertexWeight = [0.8, 0.1, 0.5, 0.1, 0.5, 1.0, 1.0, 0.1])
# NOTE: Here we can then retrieve the values
perVertWeight = cmds.getNClothPerVertex(plugName = "inputAttractPerVertex")
print(perVertWeight)
```
### All valid plugName flags
#### nCloth and nRigid
* bouncePerVertex
* collideStrengthPerVertex
* fieldMagnitudePerVertex
* frictionPerVertex
* stickinessPerVertex
* thicknessPerVertex
#### Only nCloth
* tangentialDragPerVertex
* bendPerVertex
* bendAngleDropoffPerVertex
* compressionPerVertex
* dampPerVertex
* deformPerVertex
* dragPerVertex
* inputAttractPerVertex
* liftPerVertex
* massPerVertex
* restLengthScalePerVertex
* restitutionAnglePerVertex
* rigidityPerVertex
* stretchPerVertex
* wrinklePerVertex
