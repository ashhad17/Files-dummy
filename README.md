/**
 * Airport Management System Architecture
 * 
 * Three main services:
 * 1. Flight Schedule Service - Manages flight timetables
 * 2. ATC (Air Traffic Control) Service - Coordinates flights
 * 3. Ground Clearance Service - Manages ground operations
 * 
 * Communication via Kafka
 * Data persistence with MySQL
 */

/******************************************
 * COMMON DOMAIN MODELS
 ******************************************/

// Flight.java
package com.airport.common.model;

import java.time.LocalDateTime;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Flight {
    private String flightNumber;
    private String airline;
    private String origin;
    private String destination;
    private LocalDateTime scheduledDeparture;
    private LocalDateTime scheduledArrival;
    private LocalDateTime actualDeparture;
    private LocalDateTime actualArrival;
    private FlightStatus status;
    private Integer assignedGate;
}

// FlightStatus.java
package com.airport.common.model;

public enum FlightStatus {
    SCHEDULED,
    BOARDING,
    DEPARTED,
    ARRIVING,
    LANDED,
    TAXIING,
    AT_GATE,
    CANCELLED,
    DELAYED
}

// FlightEvent.java
package com.airport.common.event;

import com.airport.common.model.Flight;
import java.time.LocalDateTime;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class FlightEvent {
    private String eventId;
    private String eventType;
    private Flight flight;
    private LocalDateTime timestamp;
    private String message;
}

/******************************************
 * FLIGHT SCHEDULE SERVICE
 ******************************************/

// FlightScheduleApplication.java
package com.airport.flightschedule;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class FlightScheduleApplication {
    public static void main(String[] args) {
        SpringApplication.run(FlightScheduleApplication.class, args);
    }
}

// FlightScheduleEntity.java
package com.airport.flightschedule.entity;

import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;

@Entity
@Table(name = "flight_schedules")
@Data
public class FlightScheduleEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String flightNumber;
    private String airline;
    private String origin;
    private String destination;
    private LocalDateTime scheduledDeparture;
    private LocalDateTime scheduledArrival;
    private String status;
    private Integer assignedGate;
}

// FlightScheduleRepository.java
package com.airport.flightschedule.repository;

import com.airport.flightschedule.entity.FlightScheduleEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;

@Repository
public interface FlightScheduleRepository extends JpaRepository<FlightScheduleEntity, Long> {
    List<FlightScheduleEntity> findByScheduledDepartureBetween(LocalDateTime start, LocalDateTime end);
    List<FlightScheduleEntity> findByScheduledArrivalBetween(LocalDateTime start, LocalDateTime end);
    FlightScheduleEntity findByFlightNumber(String flightNumber);
}

// FlightScheduleService.java
package com.airport.flightschedule.service;

import com.airport.common.event.FlightEvent;
import com.airport.common.model.Flight;
import com.airport.common.model.FlightStatus;
import com.airport.flightschedule.entity.FlightScheduleEntity;
import com.airport.flightschedule.repository.FlightScheduleRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

@Service
public class FlightScheduleService {
    
    @Autowired
    private FlightScheduleRepository flightScheduleRepository;
    
    @Autowired
    private KafkaTemplate<String, FlightEvent> kafkaTemplate;
    
    private static final String FLIGHT_SCHEDULE_TOPIC = "flight-schedule-events";
    
    public List<Flight> getUpcomingDepartures(LocalDateTime start, LocalDateTime end) {
        return flightScheduleRepository.findByScheduledDepartureBetween(start, end)
                .stream()
                .map(this::mapToFlight)
                .collect(Collectors.toList());
    }
    
    public List<Flight> getUpcomingArrivals(LocalDateTime start, LocalDateTime end) {
        return flightScheduleRepository.findByScheduledArrivalBetween(start, end)
                .stream()
                .map(this::mapToFlight)
                .collect(Collectors.toList());
    }
    
    public Flight scheduleFlight(Flight flight) {
        FlightScheduleEntity entity = new FlightScheduleEntity();
        entity.setFlightNumber(flight.getFlightNumber());
        entity.setAirline(flight.getAirline());
        entity.setOrigin(flight.getOrigin());
        entity.setDestination(flight.getDestination());
        entity.setScheduledDeparture(flight.getScheduledDeparture());
        entity.setScheduledArrival(flight.getScheduledArrival());
        entity.setStatus(FlightStatus.SCHEDULED.name());
        entity.setAssignedGate(flight.getAssignedGate());
        
        flightScheduleRepository.save(entity);
        
        // Publish event to Kafka
        FlightEvent event = new FlightEvent(
                UUID.randomUUID().toString(),
                "FLIGHT_SCHEDULED",
                flight,
                LocalDateTime.now(),
                "Flight " + flight.getFlightNumber() + " scheduled"
        );
        kafkaTemplate.send(FLIGHT_SCHEDULE_TOPIC, flight.getFlightNumber(), event);
        
        return flight;
    }
    
    public Flight updateFlightStatus(String flightNumber, FlightStatus status) {
        FlightScheduleEntity entity = flightScheduleRepository.findByFlightNumber(flightNumber);
        if (entity != null) {
            entity.setStatus(status.name());
            flightScheduleRepository.save(entity);
            
            Flight flight = mapToFlight(entity);
            
            // Publish event to Kafka
            FlightEvent event = new FlightEvent(
                    UUID.randomUUID().toString(),
                    "FLIGHT_STATUS_UPDATED",
                    flight,
                    LocalDateTime.now(),
                    "Flight " + flightNumber + " status updated to " + status
            );
            kafkaTemplate.send(FLIGHT_SCHEDULE_TOPIC, flightNumber, event);
            
            return flight;
        }
        return null;
    }
    
    private Flight mapToFlight(FlightScheduleEntity entity) {
        Flight flight = new Flight();
        flight.setFlightNumber(entity.getFlightNumber());
        flight.setAirline(entity.getAirline());
        flight.setOrigin(entity.getOrigin());
        flight.setDestination(entity.getDestination());
        flight.setScheduledDeparture(entity.getScheduledDeparture());
        flight.setScheduledArrival(entity.getScheduledArrival());
        flight.setStatus(FlightStatus.valueOf(entity.getStatus()));
        flight.setAssignedGate(entity.getAssignedGate());
        return flight;
    }
}

// FlightScheduleController.java
package com.airport.flightschedule.controller;

import com.airport.common.model.Flight;
import com.airport.common.model.FlightStatus;
import com.airport.flightschedule.service.FlightScheduleService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;
import java.util.List;

@RestController
@RequestMapping("/api/schedule")
public class FlightScheduleController {
    
    @Autowired
    private FlightScheduleService flightScheduleService;
    
    @GetMapping("/departures")
    public ResponseEntity<List<Flight>> getUpcomingDepartures(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        return ResponseEntity.ok(flightScheduleService.getUpcomingDepartures(start, end));
    }
    
    @GetMapping("/arrivals")
    public ResponseEntity<List<Flight>> getUpcomingArrivals(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        return ResponseEntity.ok(flightScheduleService.getUpcomingArrivals(start, end));
    }
    
    @PostMapping
    public ResponseEntity<Flight> scheduleFlight(@RequestBody Flight flight) {
        return ResponseEntity.ok(flightScheduleService.scheduleFlight(flight));
    }
    
    @PutMapping("/{flightNumber}/status")
    public ResponseEntity<Flight> updateFlightStatus(
            @PathVariable String flightNumber,
            @RequestParam FlightStatus status) {
        Flight updatedFlight = flightScheduleService.updateFlightStatus(flightNumber, status);
        if (updatedFlight != null) {
            return ResponseEntity.ok(updatedFlight);
        }
        return ResponseEntity.notFound().build();
    }
}

// FlightScheduleConfig.java
package com.airport.flightschedule.config;

import com.airport.common.event.FlightEvent;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class FlightScheduleConfig {
    
    @Bean
    public ProducerFactory<String, FlightEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public KafkaTemplate<String, FlightEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

/******************************************
 * ATC SERVICE
 ******************************************/

// AtcApplication.java
package com.airport.atc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AtcApplication {
    public static void main(String[] args) {
        SpringApplication.run(AtcApplication.class, args);
    }
}

// AtcEntity.java
package com.airport.atc.entity;

import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;

@Entity
@Table(name = "atc_events")
@Data
public class AtcEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String flightNumber;
    private String eventType;
    private LocalDateTime timestamp;
    private String status;
    private String message;
}

// AtcRepository.java
package com.airport.atc.repository;

import com.airport.atc.entity.AtcEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface AtcRepository extends JpaRepository<AtcEntity, Long> {
    AtcEntity findTopByFlightNumberOrderByTimestampDesc(String flightNumber);
}

// AtcService.java
package com.airport.atc.service;

import com.airport.atc.entity.AtcEntity;
import com.airport.atc.repository.AtcRepository;
import com.airport.common.event.FlightEvent;
import com.airport.common.model.Flight;
import com.airport.common.model.FlightStatus;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.UUID;

@Service
public class AtcService {
    
    @Autowired
    private AtcRepository atcRepository;
    
    @Autowired
    private KafkaTemplate<String, FlightEvent> kafkaTemplate;
    
    private static final String FLIGHT_SCHEDULE_TOPIC = "flight-schedule-events";
    private static final String ATC_TOPIC = "atc-events";
    private static final String GROUND_CLEARANCE_TOPIC = "ground-clearance-events";
    
    @KafkaListener(topics = FLIGHT_SCHEDULE_TOPIC, groupId = "atc-group")
    public void consumeFlightScheduleEvents(FlightEvent event) {
        System.out.println("Received flight schedule event: " + event);
        
        // Store the event in the database
        AtcEntity atcEvent = new AtcEntity();
        atcEvent.setFlightNumber(event.getFlight().getFlightNumber());
        atcEvent.setEventType(event.getEventType());
        atcEvent.setTimestamp(LocalDateTime.now());
        atcEvent.setStatus(event.getFlight().getStatus().name());
        atcEvent.setMessage(event.getMessage());
        atcRepository.save(atcEvent);
        
        // If it's a landing flight, notify ground clearance
        if (event.getEventType().equals("FLIGHT_STATUS_UPDATED") && 
            (event.getFlight().getStatus() == FlightStatus.ARRIVING || 
             event.getFlight().getStatus() == FlightStatus.LANDING)) {
            notifyGroundClearance(event.getFlight());
        }
    }
    
    @KafkaListener(topics = GROUND_CLEARANCE_TOPIC, groupId = "atc-group")
    public void consumeGroundClearanceEvents(FlightEvent event) {
        System.out.println("Received ground clearance event: " + event);
        
        // Store the event in the database
        AtcEntity atcEvent = new AtcEntity();
        atcEvent.setFlightNumber(event.getFlight().getFlightNumber());
        atcEvent.setEventType(event.getEventType());
        atcEvent.setTimestamp(LocalDateTime.now());
        atcEvent.setStatus(event.getFlight().getStatus().name());
        atcEvent.setMessage(event.getMessage());
        atcRepository.save(atcEvent);
        
        // If ground clearance is ready for takeoff, update flight status
        if (event.getEventType().equals("GROUND_CLEARANCE_READY_FOR_TAKEOFF")) {
            approveForTakeoff(event.getFlight());
        }
    }
    
    private void notifyGroundClearance(Flight flight) {
        FlightEvent event = new FlightEvent(
                UUID.randomUUID().toString(),
                "ATC_LANDING_ALERT",
                flight,
                LocalDateTime.now(),
                "Flight " + flight.getFlightNumber() + " is arriving, prepare for landing"
        );
        kafkaTemplate.send(GROUND_CLEARANCE_TOPIC, flight.getFlightNumber(), event);
    }
    
    public void approveForTakeoff(Flight flight) {
        // Update flight status to DEPARTED
        flight.setStatus(FlightStatus.DEPARTED);
        flight.setActualDeparture(LocalDateTime.now());
        
        FlightEvent event = new FlightEvent(
                UUID.randomUUID().toString(),
                "ATC_TAKEOFF_APPROVED",
                flight,
                LocalDateTime.now(),
                "Flight " + flight.getFlightNumber() + " is cleared for takeoff"
        );
        
        // Publish to both topics
        kafkaTemplate.send(FLIGHT_SCHEDULE_TOPIC, flight.getFlightNumber(), event);
        kafkaTemplate.send(ATC_TOPIC, flight.getFlightNumber(), event);
    }
    
    public void sendLandingApproval(Flight flight) {
        flight.setStatus(FlightStatus.LANDING);
        
        FlightEvent event = new FlightEvent(
                UUID.randomUUID().toString(),
                "ATC_LANDING_APPROVED",
                flight,
                LocalDateTime.now(),
                "Flight " + flight.getFlightNumber() + " is cleared for landing"
        );
        
        // Publish to both topics
        kafkaTemplate.send(FLIGHT_SCHEDULE_TOPIC, flight.getFlightNumber(), event);
        kafkaTemplate.send(ATC_TOPIC, flight.getFlightNumber(), event);
    }
}

// AtcController.java
package com.airport.atc.controller;

import com.airport.common.model.Flight;
import com.airport.atc.service.AtcService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/atc")
public class AtcController {
    
    @Autowired
    private AtcService atcService;
    
    @PostMapping("/landing-approval")
    public ResponseEntity<String> approveLanding(@RequestBody Flight flight) {
        atcService.sendLandingApproval(flight);
        return ResponseEntity.ok("Landing approval sent for flight " + flight.getFlightNumber());
    }
    
    @PostMapping("/takeoff-approval")
    public ResponseEntity<String> approveTakeoff(@RequestBody Flight flight) {
        atcService.approveForTakeoff(flight);
        return ResponseEntity.ok("Takeoff approval sent for flight " + flight.getFlightNumber());
    }
}

// AtcConfig.java
package com.airport.atc.config;

import com.airport.common.event.FlightEvent;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonDeserializer;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class AtcConfig {
    
    @Bean
    public ProducerFactory<String, FlightEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public ConsumerFactory<String, FlightEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "atc-group");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.airport.common.event");
        return new DefaultKafkaConsumerFactory<>(config, new StringDeserializer(),
                new JsonDeserializer<>(FlightEvent.class));
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, FlightEvent> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, FlightEvent> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
    
    @Bean
    public KafkaTemplate<String, FlightEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

/******************************************
 * GROUND CLEARANCE SERVICE
 ******************************************/

// GroundClearanceApplication.java
package com.airport.groundclearance;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GroundClearanceApplication {
    public static void main(String[] args) {
        SpringApplication.run(GroundClearanceApplication.class, args);
    }
}

// Gate.java
package com.airport.groundclearance.entity;

import jakarta.persistence.*;
import lombok.Data;

@Entity
@Table(name = "gates")
@Data
public class Gate {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private Integer gateNumber;
    private String terminal;
    private Boolean occupied;
    private String currentFlightNumber;
}

// GateRepository.java
package com.airport.groundclearance.repository;

import com.airport.groundclearance.entity.Gate;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface GateRepository extends JpaRepository<Gate, Long> {
    Gate findByGateNumber(Integer gateNumber);
    List<Gate> findByOccupied(Boolean occupied);
}

// GroundClearanceEntity.java
package com.airport.groundclearance.entity;

import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;

@Entity
@Table(name = "ground_clearance_events")
@Data
public class GroundClearanceEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String flightNumber;
    private String eventType;
    private LocalDateTime timestamp;
    private String status;
    private String message;
    private Integer gateNumber;
}

// GroundClearanceRepository.java
package com.airport.groundclearance.repository;

import com.airport.groundclearance.entity.GroundClearanceEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface GroundClearanceRepository extends JpaRepository<GroundClearanceEntity, Long> {
    List<GroundClearanceEntity> findByFlightNumberOrderByTimestampDesc(String flightNumber);
}

// GroundClearanceService.java
package com.airport.groundclearance.service;

import com.airport.common.event.FlightEvent;
import com.airport.common.model.Flight;
import com.airport.common.model.FlightStatus;
import com.airport.groundclearance.entity.Gate;
import com.airport.groundclearance.entity.GroundClearanceEntity;
import com.airport.groundclearance.repository.GateRepository;
import com.airport.groundclearance.repository.GroundClearanceRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Service
public class GroundClearanceService {
    
    @Autowired
    private GroundClearanceRepository groundClearanceRepository;
    
    @Autowired
    private GateRepository gateRepository;
    
    @Autowired
    private KafkaTemplate<String, FlightEvent> kafkaTemplate;
    
    private static final String ATC_TOPIC = "atc-events";
    private static final String GROUND_CLEARANCE_TOPIC = "ground-clearance-events";
    
    @KafkaListener(topics = ATC_TOPIC, groupId = "ground-clearance-group")
    public void consumeAtcEvents(FlightEvent event) {
        System.out.println("Received ATC event: " + event);
        
        // Store the event in the database
        GroundClearanceEntity groundClearanceEvent = new GroundClearanceEntity();
        groundClearanceEvent.setFlightNumber(event.getFlight().getFlightNumber());
        groundClearanceEvent.setEventType(event.getEventType());
        groundClearanceEvent.setTimestamp(LocalDateTime.now());
        groundClearanceEvent.setStatus(event.getFlight().getStatus().name());
        groundClearanceEvent.setMessage(event.getMessage());
        groundClearanceEvent.setGateNumber(event.getFlight().getAssignedGate());
        groundClearanceRepository.save(groundClearanceEvent);
        
        // If it's a landing approval, assign a gate
        if (event.getEventType().equals("ATC_LANDING_APPROVED")) {
            assignGate(event.getFlight());
        }
        
        // If it's a takeoff approval, update gate status
        if (event.getEventType().equals("ATC_TAKEOFF_APPROVED")) {
            releaseGate(event.getFlight().getAssignedGate());
        }
    }
    
    public void assignGate(Flight flight) {
        // Find an available gate
        List<Gate> availableGates = gateRepository.findByOccupied(false);
        if (!availableGates.isEmpty()) {
            Gate gate = availableGates.get(0);
            gate.setOccupied(true);
            gate.setCurrentFlightNumber(flight.getFlightNumber());
            gateRepository.save(gate);
            
            flight.setAssignedGate(gate.getGateNumber());
            flight.setStatus(FlightStatus.AT_GATE);
            
            // Publish event to notify ATC
            FlightEvent event = new FlightEvent(
                    UUID.randomUUID().toString(),
                    "GROUND_CLEARANCE_GATE_ASSIGNED",
                    flight,
                    LocalDateTime.now(),
                    "Flight " + flight.getFlightNumber() + " assigned to gate " + gate.getGateNumber()
            );
            kafkaTemplate.send(GROUND_CLEARANCE_TOPIC, flight.getFlightNumber(), event);
        } else {
            // No gates available, put in holding pattern
            flight.setStatus(FlightStatus.ARRIVING);
            
            FlightEvent event = new FlightEvent(
                    UUID.randomUUID().toString(),
                    "GROUND_CLEARANCE_NO_GATES",
                    flight,
                    LocalDateTime.now(),
                    "No gates available for flight " + flight.getFlightNumber()
            );
            kafkaTemplate.send(GROUND_CLEARANCE_TOPIC, flight.getFlightNumber(), event);
        }
    }
    
    public void readyForTakeoff(String flightNumber, Integer gateNumber) {
        // Find the gate
        Gate gate = gateRepository.findByGateNumber(gateNumber);
        if (gate != null && gate.getCurrentFlightNumber().equals(flightNumber)) {
            // Create a flight object
            Flight flight = new Flight();
            flight.setFlightNumber(flightNumber);
            flight.setAssignedGate(gateNumber);
            flight.setStatus(FlightStatus.BOARDING);
            
            // Notify ATC
            FlightEvent event = new FlightEvent(
                    UUID.randomUUID().toString(),
                    "GROUND_CLEARANCE_READY_FOR_TAKEOFF",
                    flight,
                    LocalDateTime.now(),
                    "Flight " + flightNumber + " is ready for takeoff from gate " + gateNumber
            );
            kafkaTemplate.send(GROUND_CLEARANCE_TOPIC, flightNumber, event);
        }
    }
    
    public void releaseGate(Integer gateNumber) {
        Gate gate = gateRepository.findByGateNumber(gateNumber);
        if (gate != null) {
            gate.setOccupied(false);
            gate.setCurrentFlightNumber(null);
            gateRepository.save(gate);
        }
    }
    
    public List<Gate> getAllGates() {
        return gateRepository.findAll();
    }
    
    public Gate getGateStatus(Integer gateNumber) {
        return gateRepository.findByGateNumber(gateNumber);
    }
}

// GroundClearanceController.java
package com.airport.groundclearance.controller;

import com.airport.groundclearance.entity.Gate;
import com.airport.groundclearance.service.GroundClearanceService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/ground")
public class GroundClearanceController {
    
    @Autowired
    private GroundClearanceService groundClearanceService;
    
    @GetMapping("/gates")
    public ResponseEntity<List<Gate>> getAllGates() {
        return ResponseEntity.ok(groundClearanceService.getAllGates());
    }
    
    @GetMapping("/gates/{gateNumber}")
    public ResponseEntity<Gate> getGateStatus(@PathVariable Integer gateNumber) {
        Gate gate = groundClearanceService.getGateStatus(gateNumber);
        if (gate != null) {
            return ResponseEntity.ok(gate);
        }
        return ResponseEntity.notFound().build();
    }
    
    @PostMapping("/ready-for-takeoff")
    public ResponseEntity<String> readyForTakeoff(
            @RequestParam String flightNumber,
            @RequestParam Integer gateNumber) {
        groundClearanceService.readyForTakeoff(flightNumber, gateNumber);
        return ResponseEntity.ok("Flight " + flightNumber + " ready for takeoff notification sent");
    }
}

// GroundClearanceConfig.java (completing the partial file)
package com.airport.groundclearance.config;

import com.airport.common.event.FlightEvent;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonDeserializer;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class GroundClearanceConfig {
    
    @Bean
    public ProducerFactory<String, FlightEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public ConsumerFactory<String, FlightEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "ground-clearance-group");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.airport.common.event");
        return new DefaultKafkaConsumerFactory<>(config, new StringDeserializer(),
                new JsonDeserializer<>(FlightEvent.class));
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, FlightEvent> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, FlightEvent> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
    
    @Bean
    public KafkaTemplate<String, FlightEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

/******************************************
 * DATABASE CONFIGURATION
 ******************************************/

// FlightScheduleDbConfig.java
package com.airport.flightschedule.config;

import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@Configuration
@EntityScan("com.airport.flightschedule.entity")
@EnableJpaRepositories("com.airport.flightschedule.repository")
public class FlightScheduleDbConfig {
}

// AtcDbConfig.java
package com.airport.atc.config;

import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@Configuration
@EntityScan("com.airport.atc.entity")
@EnableJpaRepositories("com.airport.atc.repository")
public class AtcDbConfig {
}

// GroundClearanceDbConfig.java
package com.airport.groundclearance.config;

import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@Configuration
@EntityScan("com.airport.groundclearance.entity")
@EnableJpaRepositories("com.airport.groundclearance.repository")
public class GroundClearanceDbConfig {
}

/******************************************
 * APPLICATION PROPERTIES
 ******************************************/

// flight-schedule-service/src/main/resources/application.properties
/**
server.port=8081
spring.application.name=flight-schedule-service

# Database configuration
spring.datasource.url=jdbc:mysql://localhost:3306/flight_schedule_db
spring.datasource.username=root
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect

# Kafka configuration
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
*/

// atc-service/src/main/resources/application.properties
/**
server.port=8082
spring.application.name=atc-service

# Database configuration
spring.datasource.url=jdbc:mysql://localhost:3306/atc_db
spring.datasource.username=root
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect

# Kafka configuration
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=atc-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=com.airport.common.event
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
*/

// ground-clearance-service/src/main/resources/application.properties
/**
server.port=8083
spring.application.name=ground-clearance-service

# Database configuration
spring.datasource.url=jdbc:mysql://localhost:3306/ground_clearance_db
spring.datasource.username=root
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect

# Kafka configuration
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=ground-clearance-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=com.airport.common.event
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
*/

/******************************************
 * INITIALIZATION SCRIPT FOR GATES
 ******************************************/

// GateInitializer.java
package com.airport.groundclearance.init;

import com.airport.groundclearance.entity.Gate;
import com.airport.groundclearance.repository.GateRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class GateInitializer implements CommandLineRunner {
    
    @Autowired
    private GateRepository gateRepository;
    
    @Override
    public void run(String... args) {
        // Check if gates already exist
        if (gateRepository.count() == 0) {
            // Initialize 10 gates
            for (int i = 1; i <= 10; i++) {
                Gate gate = new Gate();
                gate.setGateNumber(i);
                gate.setTerminal("Terminal " + ((i <= 5) ? "A" : "B"));
                gate.setOccupied(false);
                gate.setCurrentFlightNumber(null);
                gateRepository.save(gate);
            }
            System.out.println("Gates initialized successfully");
        }
    }
}

/******************************************
 * BUILD FILES
 ******************************************/

// Parent pom.xml
/**
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.airport</groupId>
    <artifactId>airport-management-system</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>common</module>
        <module>flight-schedule-service</module>
        <module>atc-service</module>
        <module>ground-clearance-service</module>
    </modules>

    <properties>
        <java.version>17</java.version>
        <spring-boot.version>3.2.0</spring-boot.version>
        <spring-kafka.version>3.1.0</spring-kafka.version>
        <lombok.version>1.18.30</lombok.version>
        <mysql.version>8.0.33</mysql.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.airport</groupId>
                <artifactId>common</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
*/

// common/pom.xml
/**
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.airport</groupId>
        <artifactId>airport-management-system</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>common</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
        </dependency>
    </dependencies>
</project>
*/

// flight-schedule-service/pom.xml
/**
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.airport</groupId>
        <artifactId>airport-management-system</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>flight-schedule-service</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.airport</groupId>
            <artifactId>common</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
*/

// atc-service/pom.xml
/**
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.airport</groupId>
        <artifactId>airport-management-system</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>atc-service</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.airport</groupId>
            <artifactId>common</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
*/

// ground-clearance-service/pom.xml
/**
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.airport</groupId>
        <artifactId>airport-management-system</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>ground-clearance-service</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.airport</groupId>
            <artifactId>common</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
*/

/******************************************
 * DATABASE SETUP SCRIPTS
 ******************************************/

// mysql_setup.sql
/**
-- Create databases
CREATE DATABASE IF NOT EXISTS flight_schedule_db;
CREATE DATABASE IF NOT EXISTS atc_db;
CREATE DATABASE IF NOT EXISTS ground_clearance_db;

-- Create user
CREATE USER IF NOT EXISTS 'airport_user'@'%' IDENTIFIED BY 'password';

-- Grant privileges
GRANT ALL PRIVILEGES ON flight_schedule_db.* TO 'airport_user'@'%';
GRANT ALL PRIVILEGES ON atc_db.* TO 'airport_user'@'%';
GRANT ALL PRIVILEGES ON ground_clearance_db.* TO 'airport_user'@'%';

FLUSH PRIVILEGES;
*/

/******************************************
 * DOCKER COMPOSE FOR LOCAL DEVELOPMENT
 ******************************************/

// docker-compose.yml
/**
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: airport-mysql
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: flight_schedule_db
    ports:
      - "3306:3306"
    volumes:
      - ./mysql_setup.sql:/docker-entrypoint-initdb.d/mysql_setup.sql
    networks:
      - airport-network

  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    container_name: airport-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - airport-network

  kafka:
    image: confluentinc/cp-kafka:7.3.0
    container_name: airport-kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    networks:
      - airport-network

networks:
  airport-network:
    driver: bridge
*/

/******************************************
 * README FILE WITH SETUP INSTRUCTIONS
 ******************************************/

// README.md
/**
# Airport Management System

A Spring Boot 3 microservices-based airport management system utilizing Kafka for event-driven communication between services.

## System Architecture

The system consists of three microservices:
1. **Flight Schedule Service**: Manages flight timetables
2. **ATC Service**: Air Traffic Control for coordinating flights
3. **Ground Clearance Service**: Manages ground operations including gate assignments

Communication between these services is facilitated through Kafka topics:
- `flight-schedule-events`
- `atc-events`
- `ground-clearance-events`

## Prerequisites

- Java 17+
- Maven
- Docker & Docker Compose
- MySQL 8

## Getting Started

### Step 1: Start Infrastructure Services

```bash
docker-compose up -d
```

This will start:
- MySQL database
- Zookeeper
- Kafka broker

### Step 2: Build and Run Services

Build all services:
```bash
mvn clean install
```

Run each service in separate terminals:

```bash
# Flight Schedule Service
cd flight-schedule-service
mvn spring-boot:run

# ATC Service
cd atc-service
mvn spring-boot:run

# Ground Clearance Service
cd ground-clearance-service
mvn spring-boot:run
```

## API Endpoints

### Flight Schedule Service (Port 8081)

- `GET /api/schedule/departures?start={datetime}&end={datetime}` - Get upcoming departures
- `GET /api/schedule/arrivals?start={datetime}&end={datetime}` - Get upcoming arrivals
- `POST /api/schedule` - Schedule a new flight
- `PUT /api/schedule/{flightNumber}/status?status={status}` - Update flight status

### ATC Service (Port 8082)

- `POST /api/atc/landing-approval` - Approve landing for a flight
- `POST /api/atc/takeoff-approval` - Approve takeoff for a flight

### Ground Clearance Service (Port 8083)

- `GET /api/ground/gates` - Get status of all gates
- `GET /api/ground/gates/{gateNumber}` - Get status of a specific gate
- `POST /api/ground/ready-for-takeoff?flightNumber={flightNumber}&gateNumber={gateNumber}` - Notify when a flight is ready for takeoff

## Flight Processing Flow

1. Flight Schedule Service creates new flight entries in the system
2. ATC receives flight schedule updates via Kafka
3. When flights are incoming, ATC notifies Ground Clearance Service
4. Ground Clearance assigns gates and prepares for landing
5. After landing, Ground Clearance Service manages ground operations
6. When a flight is ready for departure, Ground Clearance notifies ATC
7. ATC approves takeoff and updates Flight Schedule

## Testing the System

You can use the included Postman collection or curl commands to test the system:

Example of scheduling a flight:
```bash
curl -X POST http://localhost:8081/api/schedule \
  -H "Content-Type: application/json" \
  -d '{
    "flightNumber": "BA123",
    "airline": "British Airways",
    "origin": "LHR",
    "destination": "JFK",
    "scheduledDeparture": "2025-03-20T13:00:00",
    "scheduledArrival": "2025-03-20T22:00:00",
    "status": "SCHEDULED"
  }'
```
*/
