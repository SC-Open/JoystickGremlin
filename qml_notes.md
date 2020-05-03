# QML Notes

This contains a collection of good to know things when working with QML and Python.

## Property Binding

One of the nice properties of QML is that it has property binding, i.e. it can be directly fed with values from any `QtObject` based instance if it exposes its values properly. This can be achieved through the following:

```{python}
from Pyside2 import QtCore, QtQML

class DemoModel(QtCore.QObject):
    
    variableChanged = QtCore.Signal()
    
    def __init__(self, parent=None)
    	super().__init__(parent)
        
        self._variable = "Test"

    def _get_variable(self) -> str:
        return self._variable
    
    def _set_variable(self, value: str) -> None:
        if value == self._variable:
            return
        
        self._variable = value
        self.variableChanged.emit()
    
    variable = QtCore.Property(
    	str,
        fget=_get_variable,
        fset=_set_variable,
        notify=variableChanged
    )
```

The above skeleton exposes the `variable` member to QML and will notify the QML side when the value of `variable` is changed. However, if the QML content representing the value of `variable` changes this is not sent back to the Python side. Apparently this two way communication is not support by QML and given this was the case back in 2010 it does not seem likely to ever be added. In order to support such a two way synchronization the QML side needs to actively updated the Python side.

```{qml}
Item {
	property DemoModel model

	TextField {
		text: model.variable
		
		onTextChanged: {
			model.variable = text
		}
	}
}
```

The above QML snippet populates the textfield with the value of the `variable` from the above model class. Any changes to the value of `variable` via Python code will notify the QML side and update the visual representation accordingly. To send changes back to the Python model instance the `onTextChanged` signal needs to added. To prevent a binding loop the `_set_variable` method needs to ensure the provided value is different from the currently stored one, as otherwise a event loop would be possible.

## Python Function Return Values for QML

It is possible to call Python functions which return a value from QML as long as these are defined as `Slot` in a `QtCore` derived class. The `gremlin.ui.backend` class is a good example of a class making use of this.

```{python}
import random

from Pyside2 import QtCore

class Backend(QtCore):
    
    def __init__(self, parent=None):
        super().__init__(parent)
   	
    @QtCore.Slot(int, int, result=int)
    def randomInt(self, min_val: int, max_val: int) -> int:
		return random.randint(min_val, max_val)
```

This allows the method to be called from within any QML file which has access to the an instance of the `Backend` class.

## Model Classes with Custom Attribute Names

Accessing data from a Python model via custom names is the more convenient then having to deal with possibly changing indices. This is readily supported by QML by specifying additional model roles in the Python model being visualized via QML.

```{python}
from typing import Any, Dict
from PySide2 import QtCore, QtQML

class ColorModel(QtCore.QAbstractListModel):
    
    roles = {
        QtCore.Qt.UserRole + 1: QtCore.QByteArray("name".encode()),
        QtCore.Qt.UserRole + 2: QtCore.QByteArray("rgb".encode()),
    }
    
    def __init__(self, parent: None):
        super().__init__(parent)
        
        self._colors = []
    
    def rowCount(self, parent: QtCore.QModelIndex=...) -> int:
        return len(self._colors)
    
    def data(self, index: QtCore.QModelIndex, role: int=...) -> Any:
		if role not in ColorModel.roles:
			raise("Invalid role specified")
        
        role_name = SimpleModel.roles[role].data().decode()
        if role_name == "name":
            return self._colors[index.row].name
       	elif role_name == "rgb":
            return self._colors[index.row].rgb

	def roleNames(self) -> Dict:
        return ColorModel.roles
```

The above example specifies a simple class which holds colors. To permit QML to access the properties, i.e. name and rgb code, of each color via name the `roles` dictionary is defined and exposed. Without this there is no way to access these properties via name.

## ListView

Frequently models will contain a list of identical items that need to be visualized. As these items might be taking up more space then the ListView component has in the UI it is capable of scrolling. To turn the ListView into a container that has a scroll bar and behaves properly, i.e. like a desktop application and not a phone app the following setup is recommended.

```{qml}
ListView {
    id: idListView
    anchors.fill: parent

    // Make it behave like a sensible scrolling container
    ScrollBar.vertical: ScrollBar {}
    flickableDirection: Flickable.VerticalFlick
    boundsBehavior: Flickable.StopAtBounds

    // Content to visualize
    model: model
    delegate: idDelegate
}

Component {
	id: idDelegate
	
	...
}
```
