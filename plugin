from maya.api import OpenMaya as om2
from maya import cmds as m_cmds

PLUGIN_NAME = "nClothPerVertexCommands"
CMD_NAME_SET_NCLOTH_PER_VERTEX = 'setNClothPerVertex'
CMD_NAME_GET_NCLOTH_PER_VERTEX = 'getNClothPerVertex'

# NOTE(fuzes): These are all the valid PerVertex Attributes we allow to query and edit
_PER_VERTEX_ATTRIBUTE_NAMES = ['thicknessPerVertex', 'bouncePerVertex', 'frictionPerVertex',
                               'dampPerVertex', 'stickinessPerVertex', 'collideStrengthPerVertex',
                               'massPerVertex', 'fieldMagnitudePerVertex', 'stretchPerVertex',
                               'compressionPerVertex', 'bendPerVertex', 'bendAngleDropoffPerVertex',
                               'restitutionAnglePerVertex', 'rigidityPerVertex', 'deformPerVertex',
                               'inputAttractPerVertex', 'restLengthScalePerVertex', 'liftPerVertex',
                               'dragPerVertex', 'tangentialDragPerVertex', 'wrinklePerVertex'
                               ]

def maya_useNewAPI():
    pass

def getMobFromName(name):
    """
    Get the MObject of the given string name.
    If the MObject does not exits returns a kNull [MObject]
    :param name: [string] name of a Maya object
    :return: [MObject]
    """

    sList = om2.MSelectionList()
    if not m_cmds.objExists(name):
        return om2.MObject()
    sList.add(name)
    return sList.getDependNode(0)

_WORLD_MESH_ATTRIBUTE_NAME = "worldMesh"
# noinspection PyArgumentList,PyArgumentList,PyArgumentList
def getNObjectMobFromMob(mob):
    """
    Tries to find a valid nObject (kNCloth or kNRigid) from the selection
    If nothing is found returns a NULL [MObject]
    :param mob: [MObject]
    :return: [MObject] either NCloth or NRigid if nothing is found the [MObject is NULL
    """

    result = om2.MObject()
    # Only DAG nodes have connections to the nClothShape
    if not mob.hasFn(om2.MFn.kDagNode):
        return result

    # Return the mob if it is the nObject we are looking for
    if mob.hasFn(om2.MFn.kNCloth) or mob.hasFn(om2.MFn.kNRigid):
        result = mob
        return result

    mfn_dag = om2.MFnDagNode(mob)
    parent = mob
    if mob.hasFn(om2.MFn.kShape):
        parent = mfn_dag.parent(0)

    # NOTE(fuzes): We loop through each child of the transform node and check if the child is a interesting object
    # if is not we check for the World Mesh attribute if that is connected to a interesting object
    # if nothing was found there we found nothing interesting
    mfn_parent_dag = om2.MFnDagNode(parent)
    for x in xrange(mfn_parent_dag.childCount()):
        # TODO(fuzes): This fails if the shape of the transform is selected
        childMob = mfn_dag.child(x)

        if not childMob.hasFn(om2.MFn.kNCloth) and not childMob.hasFn(om2.MFn.kNRigid):

            if childMob.hasFn(om2.MFn.kShape):

                mfn_child_dag = om2.MFnDagNode(childMob)
                if mfn_child_dag.hasAttribute(_WORLD_MESH_ATTRIBUTE_NAME):
                    plug = mfn_child_dag.findPlug(_WORLD_MESH_ATTRIBUTE_NAME, False)
                    if not plug.numElements():
                        continue
                    worldMeshPlug = plug.elementByPhysicalIndex(0)

                    for destinationPlug in worldMeshPlug.destinations():
                        if not destinationPlug.isNull:
                            destinationMob = destinationPlug.node()

                            if destinationMob.hasFn(om2.MFn.kNCloth) or destinationMob.hasFn(om2.MFn.kNRigid):
                                result = destinationMob
                                return result
                        else:
                            continue
            else:
                continue
        else:
            result = childMob
            return result

    return result

# noinspection PyMethodOverriding,PyMethodOverriding,PyMethodOverriding,PyMethodOverriding,PyMethodOverriding,PyArgumentList,PyArgumentList
class SetNClothPerVertexMaps(om2.MPxCommand):
    """
    Maya command to set perVertex Data on nCloth or nRigid's objects
    Flags:
    * -pn/-plugName [str] Which plug of the perVertex attribute should be set
    * -vw/-vertexWeight [list] containing the values for each vertex to be set
    """

    _prev_weights = None
    _current_weights = None
    _nObjectMobHandle = None
    _plugName = None
    _needsUndo = False

    @classmethod
    def creator(cls):
        return cls()

    @staticmethod
    def newSyntax():

        stx = om2.MSyntax()
        stx.setObjectType(om2.MSyntax.kSelectionList, 1, 1)
        stx.useSelectionAsDefault(True)

        stx.addFlag("-pn", "-plugName", om2.MSyntax.kString)

        stx.addFlag("-vw", "-vertexWeight", om2.MSyntax.kDouble)
        stx.makeFlagMultiUse("-vertexWeight")

        return stx

    @staticmethod
    def hasSyntax():
        return True

    @staticmethod
    def isUndoable():
        return True

    # noinspection PyArgumentList,PyArgumentList
    def doIt(self, *args):

        # NOTE(fuzes): Here we only gather the data for the command and then call the reDoIt() method to do the rest of the job
        argDB = om2.MArgDatabase(self.syntax(), args[0])

        if argDB.isFlagSet("-pn"):
            self._plugName = argDB.flagArgumentString("-pn", 0)
            if not self._plugName in _PER_VERTEX_ATTRIBUTE_NAMES:
                m_cmds.error("Invalid Plug Name")
                return
        else:
            m_cmds.error("plugName flag must be set")

        selList = argDB.getObjectList()
        mob = selList.getDependNode(0)
        self._nObjectMobHandle = om2.MObjectHandle(mob)

        if argDB.isFlagSet("-vw"):
            self._current_weights = om2.MDoubleArray()
            for i in xrange(argDB.numberOfFlagUses("-vw")):
                argList = argDB.getFlagArgumentList("-vw", i)
                self._current_weights.append(argList.asDouble(0))
        else:
            m_cmds.error("vertexWeight flag must be set")

        self.redoIt()

    def undoIt(self, *args):

        # NOTE(fuzes): If nothing is to be undone we simply return
        if not self._needsUndo:
            return

        nMob = getNObjectMobFromMob(self._nObjectMobHandle.object())

        mfn_dep_nCloth = om2.MFnDependencyNode(nMob)
        plug = mfn_dep_nCloth.findPlug(self._plugName, False)

        dataHandle = plug.asMDataHandle()
        data = dataHandle.data()

        # NOTE(fuzes): If there where no previous weights set we simply initialize each vertex to a weight of 0.0
        doubleArrayData = om2.MFnDoubleArrayData(data)
        if not self._prev_weights:
            for i in xrange(len(doubleArrayData)):
                doubleArrayData[i] = 0
        else:
            for i, value in enumerate(self._prev_weights):
                doubleArrayData[i] = value

    def redoIt(self,*args):

        nMob = getNObjectMobFromMob(self._nObjectMobHandle.object())
        if nMob.isNull():
            m_cmds.error("Selection has no relationship with nObjects")
            return

        mfn_dep_nCloth = om2.MFnDependencyNode(nMob)
        plug = mfn_dep_nCloth.findPlug(self._plugName, False)

        dataHandle = plug.asMDataHandle()
        data = dataHandle.data()
        isDataNull = data.isNull()

        # NOTE(Fuzes): If the Data is Null we set it ourselves, else we work on the reference
        if isDataNull:
            newData = om2.MFnDoubleArrayData()
            mobData = newData.create(self._current_weights)
            dataHandle.setMObject(mobData)
            plug.setMDataHandle(dataHandle)
            self._needsUndo = True
            return
        else:
            doubleArrayData = om2.MFnDoubleArrayData(data)
            self._prev_weights = om2.MDoubleArray()
            for i, value in enumerate(self._current_weights):
                self._prev_weights.append(doubleArrayData[i])
                doubleArrayData[i] = value
            self._needsUndo = True
            return

# noinspection PyMethodOverriding,PyArgumentList
class GetNClothPerVertex(om2.MPxCommand):
    """
    Maya command to get perVertex Data on nCloth or nRigid's objects
    Flags:
    * -pn/-plugName [str] Which plug of the perVertex attribute should be getting back
    """

    @classmethod
    def creator(cls):
        return cls()

    @staticmethod
    def newSyntax():
        stx = om2.MSyntax()
        stx.setObjectType(om2.MSyntax.kSelectionList, 1, 1)
        stx.useSelectionAsDefault(True)
        stx.addFlag("-pn", "-plugName", om2.MSyntax.kString)
        return stx

    @staticmethod
    def hasSyntax():
        return True

    def doIt(self, *args):

        # NOTE(fuzes): Here we only gather the data for the command
        argDB = om2.MArgDatabase(self.syntax(), args[0])

        if argDB.isFlagSet("-pn"):
            plugName = argDB.flagArgumentString("-pn", 0)
            if not plugName in _PER_VERTEX_ATTRIBUTE_NAMES:
                m_cmds.error("Invalid Plug Name")
                return
        else:
            m_cmds.error("plugName flag must be set")

        selList = argDB.getObjectList()
        mob = selList.getDependNode(0)
        nMob = getNObjectMobFromMob(mob)
        if nMob.isNull():
            m_cmds.error("Selection has no relationship with nObjects")
            return

        mfn_dep_nCloth = om2.MFnDependencyNode(nMob)
        plug = mfn_dep_nCloth.findPlug(plugName, False)

        dataHandle = plug.asMDataHandle()
        data = dataHandle.data()
        isDataNull = data.isNull()

        result = om2.MDoubleArray()
        if isDataNull:
            self.clearResult()
            self.setResult(result)
        else:
            doubleArrayData = om2.MFnDoubleArrayData(data)
            result = doubleArrayData.array()
            self.clearResult()
            self.setResult(result)

# noinspection PyArgumentList
def initializePlugin(mob):
    fnPlugin = om2.MFnPlugin(mob)
    fnPlugin.setName(PLUGIN_NAME)

    fnPlugin.registerCommand(CMD_NAME_SET_NCLOTH_PER_VERTEX,
                             SetNClothPerVertexMaps.creator,
                             SetNClothPerVertexMaps.newSyntax)

    fnPlugin.registerCommand(CMD_NAME_GET_NCLOTH_PER_VERTEX,
                             GetNClothPerVertex.creator,
                             GetNClothPerVertex.newSyntax
                             )

# noinspection PyArgumentList
def uninitializePlugin(mob):
    fnPlugin = om2.MFnPlugin(mob)
    fnPlugin.deregisterCommand(CMD_NAME_SET_NCLOTH_PER_VERTEX)
    fnPlugin.deregisterCommand(CMD_NAME_GET_NCLOTH_PER_VERTEX)



