# Using the ecl EKF
This tutorial answers common questions about use of the ECL EKF algorithm. 

## What is the ecl EKF?
The ECL (Estimation and Control Library) uses an Extended Kalman Filter algorithm to processe sensor measurements and provide an estimate of the following states:

* Quaternion defining the rotation from earth to body frame
* Velocity at the IMU North,East,Down (m/s)
* Position at the IMU North,East,Down (m)
* IMU delta angle bias estimates X,Y,Z (rad)
* IMU delta velocity bias estimates X,Y,Z(m/s)
* Earth Magnetic field components North, East,Down (gauss)
* Vehicle body frame magnetic field bias X,Y,Z (gauss)
* Wind velocity North,East (m/s)

The EKF runs on a delayed 'fusion time horizon' to allow for different time delays on each measurement relative to the IMU. Data for each sensor is FIFO buffered and retrieved from the buffer by the EKF to be used at the correct time. The delay compensation for each sensor is controlled by the EKF2_*_DELAY parameters.

A complementary filter is used to propagate the states forward from the 'fusion time horizon' to current time using the buffered IMU data. The time constant for this filter is controlled by the EKF2_TAU_VEL and EKF2_TAU_POS parameters.

Note: The 'fusion time horizon' delay and length of the buffers is determined by the largest of the EKF2_*_DELAY parameters. If a sensor is not being used, it is recommended to set its time delay to zero. Reducing the 'fusion time horizon' delay reduces errors in the complementary filter used to propagate states forward to current time.

The position and velocity states are adjusted to account for the offset between the IMU and the body frame before they are output to the control loops. The position of the IMU relative to the body frame is set by the EKF2_IMU_POS_X,Y,Z parameters.

The EKF uses the IMU data for state prediction only. IMU data is not used as an observation in the EKF derivation. The algebraic equations for the covariance prediction, state update and covariance update were derived using the Matlab symbolic toolbox and can be found here: [Matlab Symbolic Derivation](https://github.com/PX4/ecl/blob/master/matlab/scripts/Inertial%20Nav%20EKF/GenerateNavFilterEquations.m)

## What sensor measurements does it use?
The EKF has different modes of operation that allow for different combinations of sensor measurements:

On start-up the filter checks for a minimum viable combination of sensors and after initial tilt, yaw and height alignment is completed, enters a mode that provides rotation, vertical velocity,  vertical position, IMU delta angle bias and IMU delta velocity bias estimates. The IMU along with a source of yaw (magnetometer or external vision) and a source of height data are required for all EKF modes of operation.

###IMU
* Three axis body fixed Inertial Measurement unit delta angle and delta velocity data at a minimum rate of 100Hz. Note: Coning corrections should be applied to the IMU delta angle data before it is used by the EKF.

###Magnetometer
Three axis body fixed magnetometer data OR external vision system pose data at a minimum rate of 5Hz is required. Magnetometer data can be used in two ways:

* Magnetometer measurements are converted to a yaw angle using the tilt estimate and magnetic declination. This yaw angle is then used as an observation by the EKF. This method is less accurate and does not allow for learning of body frame field offsets, however it is more robust to magnetic anomalies and large start-up gyro biases. It is the default method used during start-up and on ground.
* The  XYZ magnetometer readings are used as separate observations. This method is more accurate and allows body frame offsets to be learned, but assumes the earth magnetic field environment only changes slowly and performs less well when there are significant external magnetic anomalies. It is the default method when the vehicle is airborne and has climbed past 1.5 m altitude.

The logic used to select the mode is set by the EKF2_MAG_TYPE parameter.

###Height
A source of height data - either GPS, barometric pressure, range finder or external vision at a minimum rate of 5Hz is required. Note: The primary source of height data is controlled by the EKF2_HGT_MODE parameter. 

If these measurements are not present, the EKF will not start. When these measurements have been detected, the EKF will initialise the states and complete the tilt and yaw alignment. When tilt and yaw alignment is complete, the EKF can then transition to other modes of operation  enabling use of additional sensor data:

###GPS
GPS measurements will be used for position and velocity if the following conditions are met:

* GPS use is enabled via setting of the EKF2_AID_MASK parameter.
* GPS quality checks have passed. These checks are controlled by the EKF2_GPS_CHECK and EKF2_REQ<> parameters. 
* GPS height can be used directly by the EKF via setting of the EKF2_HGT_MODE parameter.

###Range Finder
Range finder distance to ground is used a by a single state filter to estimate the vertical position of the terrain relative to the height datum. 

If operating over a flat surface that can be used as a zero height datum, the range finder data can also be used directly by the EKF to estimate height by setting the EKF2_HGT_MODE parameter to 2. 

###Airspeed
Equivalent Airspeed (EAS) data can be used to estimate wind velocity and reduce drift when GPS is lost by setting EKF2_ARSP_THR to a positive value. Airspeed data will be used when it exceeds the threshold set by a positive value for EKF2_ARSP_THR and the vehicle type is not rotary wing.

###Synthetic Sideslip
Fixed wing platforms can take advantage of an assumed sidelsip observation of zero to improve wind speed estimation and also enable wind speed estimation without an airspeed sensor. This is enabled by setting the EKF2_FUSE_BETA parameter to 1.

###Optical Flow
Optical flow data will be used if the following conditions are met:
 * Valid range finder data is available.
 * Bit position 1 in the EKF2_AID_MASK parameter is true.
 * The quality metric returned by the flow sensor is greater than the minimum requirement set by the EKF2_OF_QMIN parameter

###External Vision
* External vision system horizontal position data will be used if bit position 3 in the EKF2_AID_MASK parameter is true.
* External vision system vertical position data will be used if the EKF2_HGT_MODE parameter is set to 3.
* External vision system pose data will be used for yaw estimation if bit position 4 in the EKF2_AID_MASK parameter is true.

## How do I use the 'ecl' library EKF?
Set the SYS_MC_EST_GROUP parameter to 2 to use the ecl EKF.

## What are the advantages and disadvantages of the ecl EKF over other estimators?
Like all estimators, much of the performance comes from the tuning to match sensor characteristics. Tuning is a compromise between accuracy and robustness and although we have attempted to provide a tune that meets the needs of most users, there will be applications where tuning changes are required. 

For this reason, no claims for accuracy relative to the legacy combination of attitude_estimator_q + local_position_estimator have been made and the best choice of estimator will depend on the application and tuning.

### Disadvantages
* The ecl EKF is a complex algorithm that requires a good understanding of extended Kalman filter theory and its application to navigation problems to tune successfully. It is therefore more difficult for users that are not achieving good results to know what to change.
* The ecl EKF uses more RAM and flash space
* The ecl EKF uses more logging space.
* The ecl EKF has had less flight time
 
### Advantages
* The ecl EKF is able to fuse data from sensors with different time delays and data rates in a mathematically consistent way which improves accuracy during dynamic manoeuvres once time delay parameters are set correctly.
* The ecl EKF is capable of fusing a large range of different sensor types.
* The ecl EKF detects and reports statistically significant inconsistencies in sensor data helping to diagnose sensor issues.
* For fixed wing operation, the ecl EKF estimates wind speed with or without an airspeed sensor and is able to use the estimated wind in combination with airspeed measurements and sideslip assumptions to extend the dead-reckoning time avalable if GPS is lost in flight.
* The ecl EKF estimates 3-axis accelerometer bias which improves accuracy for tailsitters and other vehicles that experience large attitude changes between flight phases.
* The federated architecture (combined attitude and position/velocity estimation) means that attitude estimation benefits from all sensor measurements. This should provide the potential for improved attitude estimation if tuned correctly. 

## How do I check the EKF performance?
EKF outputs, states and status data are published to a number of uORB topics which are logged to the SD card during flight.

###Output Data

* Attitude output data: Refer to vehicle_attitude.msg for definitions.
* Local position output data: Refer to vehicle_local_position.msg for definitions.
* Control loop feedback data: Refer to control_state.msg for definitions.
* Global (WGS-84) output data: Refer to vehicle_global_position.msg for definitions.
* Wind velocity output data: Refer to wind_estimate.msg for definitions.

###States

Refer to states[32] in estimator_status message. The index map for states[32] is as follows:

* [0 ... 3] Quaternions
* [4 ... 6] Velocity NED (m/s)
* [7 ... 9] Position NED (m)
* [10 ... 12] IMU delta angle bias XYZ (rad)
* [13 ... 15] IMU delta velocity bias XYZ (m/s)
* [16 ... 18] Earth magnetic field NED (gauss)
* [19 ... 21] Body magnetic field XYZ (gauss)
* [22 ... 23] Wind velocity NE (m/s)
* [24 ... 32] Not Used

###State Variances
Refer to covariances[28] in the estimator_status message. The index map for covariances[28] is as follows:

* [0 ... 3] Quaternions
* [4 ... 6] Velocity NED (m/s)^2
* [7 ... 9] Position NED (m^2)
* [10 ... 12] IMU delta angle bias XYZ (rad^2)
* [13 ... 15] IMU delta velocity bias XYZ (m/s)^2
* [16 ... 18] Earth magnetic field NED (gauss^2)
* [19 ... 21] Body magnetic field XYZ (gauss^2)
* [22 ... 23] Wind velocity NE (m/s)^2
* [24 ... 28] Not Used

###Observation Innovations

* Magnetometer XYZ (gauss) : Refer to mag_innov[3] in the ekf2_innovations message.
* Yaw angle (rad) : Refer to heading_innov in the ekf2_innovations message.
* Velocity and position innovations : Refer to vel_pos_innov[6] in the ekf2_innovations. The index map for vel_pos_innov[6] is as follows:
  * [0 ... 2] Velocity NED (m/s)
  * [3 ... 5] Position NED (m)
* True Airspeed (m/s) : Refer to airspeed_innov in the ekf2_innovations message.
* Synthetic sideslip (rad) : Refer to beta_innov in the ekf2_innovations message.
* Optical flow XY (rad/sec) : Refer to flow_innov in the ekf2_innovations message.
* Height above ground (m) : Refer to hagl_innov in the ekf2_innovations message.

###Observation Innovation Variances

* Magnetometer XYZ (gauss^2) : Refer to mag_innov_var[3] in the ekf2_innovations message.
* Yaw angle (rad^2) : Refer to heading_innov_var in the ekf2_innovations message.
* Velocity and position innovations : Refer to vel_pos_innov_var[6] in the ekf2_innovations. The index map for vel_pos_innov_var[6] is as follows:
  * [0 ... 2] Velocity NED (m/s)^2
  * [3 ... 5] Position NED (m^2)
* True Airspeed (m/s)^2 : Refer to airspeed_innov_var in the ekf2_innovations message.
* Synthetic sideslip (rad^2) : Refer to beta_innov_var in the ekf2_innovations message.
* Optical flow XY (rad/sec)^2 : Refer to flow_innov_var in the ekf2_innovations message.
* Height above ground (m^2) : Refer to hagl_innov_var in the ekf2_innovations message.

###Output Complementary Filter
The output complementary filter is used to propagate states forward from the fusion time horizon to current time. To check the magnitude of the angular, velocity and position tracking errors measured at the fusion time horizon, refer to output_tracking_error[3] in the ekf2_innovations message. The index map is as follows:

* [0] Angular tracking error magnitude (rad)
* [1] Velocity tracking error magntiude (m/s). The velocity tracking time constant can be adjusted using the EKF2_TAU_VEL parameter. Reducing this parameter reduces steady state errors but increases the amount of observation noise on the NED velocity outputs.
* [2] Position tracking error magntiude (m). The position tracking time constant can be adjusted using the EKF2_TAU_POS parameter. Reducing this parameter reduces steady state errors but increases the amount of observation noise on the NED position outputs.

###EKF Errors
The EKF constains internal error checking for badly conditioned state and covariance updates. Refer to the filter_fault_flags in the estimator_status message.

###Observation Errors
There are two categories of observation faults:

* Loss of data. An example of this is a range finder failing to provide a return.
* The innovation, which is the difference between the state prediction and sensor observation is excessive. An example of this is excessive vibration causing a large vertical position error, resulting in the barometer height measurement being rejected.

Both of these can result in observation data being rejected for long enough to cause the EKF to attempt a reset of the states using the sensor observations. All observations have a statistical confidence check applied to the innovations. The number of standard deviations for the check are controlled by the EKF2_<>_GATE parameter for each observation type.

Test levels are  available in the estimator_status message as follows:

* mag_test_ratio : ratio of the largest magnetometer innovation component to the innovation test limit
* vel_test_ratio : ratio of the largest velocity innovation component to the innovation test limit
* pos_test_ratio : ratio of the largest horizontal position innovation component to the innovation test limit
* hgt_test_ratio : ratio of the vertical position innovation to the innovation test limit
* tas_test_ratio : ratio of the true airspeed innovation to the innovation test limit
* hagl_test_ratio : ratio of the height above ground innovation to the innovation test limit

For a binary pass/fail summary for each sensor, refer to innovation_check_flags in the estimator_status message.

##What should I do if the height estimate is diverging?
The most common cause of EKF height diverging away from GPS and altimeter measurements during flight is clipping and/or aliasing of the IMU measurements caused by vibration. If this is occurring, then the following signs should be evident in the data

1. ekf2_innovations.vel_pos_innov[3] and  ekf2_innovations.vel_pos_innov[5] will both have the same sign.
2. estimator_status.hgt_test_ratio will be greater than 1.0

The recommended first step is to  esnure that the autopilot is isolated from the airframe using an effective isolatoin mounting system. An isolaton mount has 6 degrees of freedom, and therefore 6 resonant frequencies. As a general rule, the 6 resonant frequencies of the autopilot on the isolation mount should be above 25Hz to avoid interaction with the autopilot dynamics and below the frequency of the motors.

An isolation mount can make vibration worse if the resonant frequncies coincide with motor or propeller blade passage frequencies.

The EKF can be made more resistant to vibration induced height divergence by making the following parameter changes:

1. Double the value of the innovation gate for the primary height sensor. If using barometeric height this is EK2_EKF2_BARO_GATE.
2. Increase the value of EKF2_ACC_NOISE to 0.5 initially. If divergence is still occurring,   increase in further increments of 0.1 but do not go above 1.0

Note that the effect of these changes will make the EKF more sensitive to errors in GPS vertical velocity and barometric pressure.

##What should I do if the position estimate is diverging?
The most common causes of position divergence are:

* High vibration levels. 
 * Fix by improving mechanical isolation of the autopilot. Increasing the value of EKF2_ACC_NOISE can help, but does make the EKF more vulnerable to GPS glitches
* Large gyro bias offsets. 
 * Fix by re-calibrating the gyro. Check for excessive temperature sensitivity (> 3 deg/sec bias change during warm-up from a cold start and replace the sensor if affected of insulate to to slow the rate of temeprature change.
* Bad yaw alignment
  * Check the magntometer calibration and alignment.
  * Check the heading shown QGC is within within 15 deg truth
* Poor GPS accuracy
 * Check for interference
 * Improve separation and shielding
 * Check flying location for GPS signal obstructions and reflectors (nearboy tall buildings)
* Loss of GPS

Determining which of these is the primary casue requires a methodical approach to analysis of the EKF log data:

* Plot the velocty innovation test ratio - estimator_status.vel_test_ratio
* Plot the horizontal position innovation test ratio - estimator_status.pos_test_ratio
* Plot the height innovation test ratio - estimator_status.hgt_test_ratio
* Plot the magnetoemrer innovation test ratio - estimator_status.mag_test_ratio
* Plot the GPS receier reported speed accuracy - vehicle_gps_position.s_variance_m_s
* Plot the IMU delta angle state estimates - estimator_status.states[10], states[11] and states[12]
* Plot the EKF internal high frequency vibration metrics:
  * Delta angle coning vibration - estimator_status.vibe[0]
  * High frequency delta angle vibration - estimator_status.vibe[1]
  * High frequency delta velocity vibration - estimator_status.vibe[2]

During normal operation, all the test ratios should remain below 0.5 with only occasional spikes above this as shown in the example below from a successful flight:
![Position, Velocity, Height and Magnetometer Test Ratios](Screen Shot 2016-12-02 at 9.20.50 pm.png)
The following plot shows the EKF vibration metrics for a multirotor with good isolation. The landing shock and the increased vibration during takeoff and landing can be seen. Insifficient data has been gathered with these metrics to provide specific advice on maximum thresholds.
![](Screen Shot 2016-12-02 at 10.24.00 pm.png)
The above vibration metrics are of limited value as the presence of vibration at a frequency close to the IMU sampling frequency (1kHz for most boards) will cause  offsets to appear in the data that do not show up in the high frequency vibration metrics. The only way to detect aliasing errors is in their effect on inertial navigation accuracy and the rise in innovation levels.

In addition to generating large position and velocity test ratios of > 1.0, the different error mechanisms affect the other test ratios in different ways:

* High vibration levels normally affect vertical positiion and velocity innovations as well as the horizontal componenets. Magnetometer test levels are only affected to a small extent.

(insert example plots showing bad vibration here)
* Large gyro bias offsets are normally characterised by a change in the value of delta angle bias greater than 5E-4 during flight (equivalent to ~3 deg/sec) and can also cause a large increase in the magnetometer test ratio if the yaw axis is affected. Height is normally unaffected other than extreme cases. Switch on bias value of up to 5 deg/sec can be tolerated provided the filter is given time time settle before flying . Pre-flight checks performed by the commander should prevent arming if the position is diverging.

(insert example plots showing bad gyro bias here)
* Bad yaw alignment causes a velocity test ratio that increases rapidly when the vehicle starts moving due inconsistency in the direction of velocity calculatde by the inertial nav and the  GPS measurement. Magnetometer innovations are slightly affected. Height is normally unaffected. 

(insert example plots showing bad yaw alignment here)
* Poor GPS accuracy is normally accompanied by a rise in the reported velocity error of the receiver.

(insert example plots showing bad GPS data here)
* Loss of GPS data will be shown by the velocity and position innvoation test ratios 'flat-lining'. If this occurs, check the oher GPS status data in vehicle_gps_position for further information.

(insert example plosts showing loss of GPS data here)
