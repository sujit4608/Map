# Map
<br>
how are you
++ maphandeler.h
#ifndef MAPHANDLER_H
#define MAPHANDLER_H

#include <QObject>
#include <QVariantList>

class MapHandler : public QObject {
    Q_OBJECT
    Q_PROPERTY(QVariantList locations READ locations NOTIFY locationsChanged)
    Q_PROPERTY(QVariantList pathPoints READ pathPoints NOTIFY pathChanged)

public:
    explicit MapHandler(QObject *parent = nullptr);

    QVariantList locations() const;
    QVariantList pathPoints() const;

    Q_INVOKABLE void addLocation(double latitude, double longitude);
    Q_INVOKABLE void moveLocation(int index, double latitude, double longitude);
    Q_INVOKABLE void addToPath(int index);

signals:
    void locationsChanged();
    void pathChanged();

private:
    QVariantList m_locations;
    QVariantList m_pathPoints;
};

#endif // MAPHANDLER_H

++ maphanderler.cpp

#include "MapHandler.h"

MapHandler::MapHandler(QObject *parent) : QObject(parent) {}

QVariantList MapHandler::locations() const {
    return m_locations;
}

QVariantList MapHandler::pathPoints() const {
    return m_pathPoints;
}

void MapHandler::addLocation(double latitude, double longitude) {
    QVariantMap location;
    location["latitude"] = latitude;
    location["longitude"] = longitude;
    m_locations.append(location);
    emit locationsChanged();
}

void MapHandler::moveLocation(int index, double latitude, double longitude) {
    if (index >= 0 && index < m_locations.size()) {
        m_locations[index].toMap()["latitude"] = latitude;
        m_locations[index].toMap()["longitude"] = longitude;
        emit locationsChanged();
        emit pathChanged(); // Update path when a marker moves
    }
}

void MapHandler::addToPath(int index) {
    if (index >= 0 && index < m_locations.size()) {
        m_pathPoints.append(m_locations[index]);
        emit pathChanged();
    }
}

++main.qml
import QtQuick 2.15
import QtQuick.Controls 2.15
import QtLocation 5.15
import QtPositioning 5.15

ApplicationWindow {
    visible: true
    width: 800
    height: 600
    title: "Qt Map with Draggable Markers and Path"

    Map {
        id: map
        anchors.fill: parent
        plugin: Plugin { name: "osm" }
        center: QtPositioning.coordinate(18.5204, 73.8567) // Pune
        zoomLevel: 13

        MouseArea {
            anchors.fill: parent
            onClicked: {
                mapHandler.addLocation(map.toCoordinate(Qt.point(mouse.x, mouse.y)).latitude,
                                       map.toCoordinate(Qt.point(mouse.x, mouse.y)).longitude));
            }
        }

        Repeater {
            model: mapHandler.locations
            delegate: MapQuickItem {
                id: marker
                coordinate: QtPositioning.coordinate(modelData.latitude, modelData.longitude)
                anchorPoint.x: icon.width / 2
                anchorPoint.y: icon.height / 2
                sourceItem: Image {
                    id: icon
                    source: "qrc:/marker.png"
                    width: 30
                    height: 30
                    smooth: true
                }

                property bool dragging: false

                MouseArea {
                    id: dragArea
                    anchors.fill: parent
                    drag.target: parent
                    onPressed: marker.dragging = true
                    onReleased: {
                        marker.dragging = false
                        var newCoord = map.toCoordinate(Qt.point(icon.x + icon.width / 2, 
                                                                 icon.y + icon.height / 2));
                        mapHandler.moveLocation(index, newCoord.latitude, newCoord.longitude);
                    }
                    onPositionChanged: {
                        if (marker.dragging) {
                            var newCoord = map.toCoordinate(Qt.point(icon.x + icon.width / 2, 
                                                                     icon.y + icon.height / 2));
                            marker.coordinate = newCoord;
                        }
                    }
                }

                MouseArea {
                    anchors.fill: parent
                    onClicked: {
                        mapHandler.addToPath(index); // Add marker to path when clicked
                    }
                }
            }
        }

        MapPolyline {
            id: pathLine
            line.width: 3
            line.color: "blue"
            smooth: true
            path: ListModel {}

            Component.onCompleted: {
                mapHandler.pathChanged.connect(function() {
                    pathLine.path.clear();
                    for (var i = 0; i < mapHandler.pathPoints.length; i++) {
                        var loc = mapHandler.pathPoints[i];
                        pathLine.path.append({ latitude: loc.latitude, longitude: loc.longitude });
                    }
                });
            }
        }
    }
}

++ main.cpp

#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include "MapHandler.h"

int main(int argc, char *argv[]) {
    QGuiApplication app(argc, argv);
    QQmlApplicationEngine engine;

    // Register the C++ class with QML
    MapHandler mapHandler;
    engine.rootContext()->setContextProperty("mapHandler", &mapHandler);

    engine.load(QUrl(QStringLiteral("qrc:/main.qml")));

    if (engine.rootObjects().isEmpty())
        return -1;

    return app.exec();
}
