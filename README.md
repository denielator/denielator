#include "DetectorConstruction.hh"
#include "G4Pow.hh"
#include "G4RunManager.hh"
#include "G4NistManager.hh"
#include "G4Box.hh"
#include "G4Tubs.hh"
#include "G4Paraboloid.hh"
#include "G4LogicalVolume.hh"
#include "G4PVPlacement.hh"
#include "G4SystemOfUnits.hh"

namespace B1
{

G4VPhysicalVolume* DetectorConstruction::Construct()
{
  G4bool checkOverlaps = true;

  G4NistManager* nist = G4NistManager::Instance();
  G4double world_sizeXY = 20 *m;
  G4double world_sizeZ = 20 *m;
  G4Material* world_mat = nist->FindOrBuildMaterial("G4_AIR");

  G4Material* LS = new G4Material("LS", 0.963 * g / cm3, 2);
  LS->AddElement(nist->FindOrBuildElement("C"), 14);
  LS->AddElement(nist->FindOrBuildElement("H"), 10);

  auto solidWorld = new G4Box("World", world_sizeXY, world_sizeXY, world_sizeZ);
  auto logicWorld = new G4LogicalVolume(solidWorld, world_mat, "World");
  auto physWorld = new G4PVPlacement(nullptr, G4ThreeVector(), logicWorld, "World", nullptr, false, 0, checkOverlaps);

  // Define the parabolic antenna
  G4double antennaRadius = 50 * cm;
  G4double antennaHeight = 10 * cm;
  G4double antennaFocalLength = antennaRadius / 2;
  G4Paraboloid* solidAntenna = new G4Paraboloid("Antenna", antennaRadius, antennaHeight);
auto logicAntenna = new G4LogicalVolume(solidAntenna, world_mat, "LogicalAntenna");
new G4PVPlacement(nullptr, G4ThreeVector(), logicAntenna, "Antenna", logicWorld, false, 0, checkOverlaps);

  // Scintillator liquid cylinders
  const int nCylinders = 64;
  const G4double cylinderRadius = 38 *mm;
  const G4double cylinderLength = 51 *mm;
  const G4double cylinderDistance = 50 *cm;

  // Place the scintillator detectors on the parabolic antenna
  for (int i = 1; i <= nCylinders; i++) {
    G4double theta = 2 * CLHEP::pi * (i - 1) / nCylinders;
    G4double radius = antennaRadius;
    G4double x = radius * cos(theta);
    G4double y = radius * sin(theta);
    G4double z = antennaFocalLength - sqrt(radius * radius - x * x - y * y);

    auto solidScintillator = new G4Tubs("solidScintillator", 0 * mm, cylinderRadius, 0.5 * cylinderLength, 0 * deg, 360 * deg);
    auto logicScintillator = new G4LogicalVolume(solidScintillator, LS, "logicalScintillator");
    new G4PVPlacement(G4Translate3D(x, y, z) * G4Rotate3D(theta, G4ThreeVector(0, 0, 1)), logicScintillator, "Scintillator", logicWorld, false, i, checkOverlaps);

    // Detector sensitive volume at the end of the scintillator cylinder
    auto solidDetector = new G4Tubs("solidDetector", 0 * mm, cylinderRadius, 0.5 * cylinderLength, 0 * deg, 360 * deg);
    auto logicDetector = new G4LogicalVolume(solidDetector, world_mat, "logicalDetector");
    new G4PVPlacement(G4Translate3D(x, y, z) * G4Rotate3D(theta, G4ThreeVector(0, 0, 1)), logicDetector, "Detector", logicWorld, false, i, checkOverlaps);
  }

  return physWorld;
}

} // namespace B1
